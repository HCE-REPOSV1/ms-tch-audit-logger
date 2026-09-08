# ms-tch-audit-logger

> Audit Logger Service generado por **Jarvis Platform** — 2/4/2026

Consumidor Kafka que persiste eventos de auditoría en SQL Server.
Expone endpoints HTTP de consulta protegidos con API key.

## Modelo de datos

```
AppUser       — copia denormalizada del usuario (upsert en cada LOGIN_SUCCESS)
AuthSession   — sesiones de autenticación (FK user_id → AppUser.user_id)
AuthToken     — tokens emitidos por sesión (FK session_id → AuthSession.session_id)
AuditEvent    — registro central de todos los eventos ← tabla principal
AuditTrace    — trazas distribuidas entre microservicios
```

`AuditEvent.session_id` es `uniqueidentifier` (consistente con `AuthSession`/`AuthToken`)
pero **sin FK** hacia `AuthSession`: es la tabla de mayor volumen de escritura y de
retención regulatoria más larga — una FK síncrona acoplaría su latencia de escritura
a validación referencial, y `AuthSession` es rotativa/purgable.

## Routing de eventos Kafka

| event_type | AuditEvent | AppUser | AuthSession | AuthToken | AuditTrace |
|------------|:-:|:-:|:-:|:-:|:-:|
| LOGIN_SUCCESS | ✓ | upsert | crear | — | — |
| LOGIN_FAILED  | ✓ | — | — | — | — |
| LOGOUT        | ✓ | — | status=revoked | — | — |
| TOKEN_REFRESH | ✓ | — | — | crear | — |
| GATEWAY_REQUEST | ✓ | — | — | — | upsert |

## Endpoints HTTP

Las rutas de `/audit/...` van con el prefijo de versión (`app.setGlobalPrefix('api')` +
`enableVersioning` en `main.ts`, `defaultVersion: '1'`) → `/api/v1/audit/...`. `/health` está
excluido del prefijo `api` (`exclude: ['health']`) y no lleva versión.

Todos los endpoints bajo `/api/v1/audit/...` — **incluido** `/api/v1/audit/health`, el guard
(`ApiKeyGuard`) se aplica a nivel de controller sin excepción por ruta — requieren el header
`x-api-key` cuando `AUDIT_API_KEY` tiene valor configurado; si está vacío, el guard es permisivo
(modo desarrollo).

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/health` | Health check global del servicio (fuera de `/audit`, sin API key, sin prefijo de versión) |
| GET | `/api/v1/audit/events` | Consultar eventos con filtros |
| GET | `/api/v1/audit/trace/:traceId` | Traza completa — todos los eventos de un request |
| GET | `/api/v1/audit/session/:sessionId` | Sesión + eventos + tokens |
| GET | `/api/v1/audit/health` | Health check específico del módulo de auditoría |

### Autenticación — API Key

Los endpoints de consulta están protegidos con una clave estática configurada en `.env`:

```bash
# .env del ms-tch-audit-logger
AUDIT_API_KEY=mi-clave-secreta-interna

# Uso en cada request
x-api-key: mi-clave-secreta-interna
```

- Si `AUDIT_API_KEY` está **vacío**: el guard es permisivo (útil en desarrollo)
- Si `AUDIT_API_KEY` tiene valor: cualquier request sin el header o con clave incorrecta recibe `401 Unauthorized`
- En producción **siempre** configurar un valor largo y aleatorio

### Filtros disponibles en GET /api/v1/audit/events

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `userId` | string | Filtrar por UUID de usuario |
| `username` | string | Filtrar por código de usuario |
| `eventType` | string | `LOGIN_SUCCESS`, `LOGIN_FAILED`, `LOGOUT`, `TOKEN_REFRESH`, `GATEWAY_REQUEST` |
| `outcome` | string | `SUCCESS`, `FAILED`, `ERROR` |
| `sourceSystem` | string | Sistema origen del evento |
| `traceId` | string | UUID de traza |
| `from` | ISO 8601 | Fecha/hora desde |
| `to` | ISO 8601 | Fecha/hora hasta |
| `limit` | number | Máximo de resultados (default 200) |

### Ejemplos de consulta

```bash
# Variable de entorno con la key (o sustituir directamente)
KEY=mi-clave-secreta-interna

# Todos los logins fallidos
curl -H "x-api-key: $KEY" "http://localhost:10400/api/v1/audit/events?eventType=LOGIN_FAILED"

# Logins fallidos en un rango de fechas
curl -H "x-api-key: $KEY" "http://localhost:10400/api/v1/audit/events?eventType=LOGIN_FAILED&from=2026-01-01T00:00:00Z&to=2026-01-31T23:59:59Z"

# Seguir un request a través de todos los microservicios
curl -H "x-api-key: $KEY" "http://localhost:10400/api/v1/audit/trace/abc-123-uuid"

# Ver sesión completa de un usuario
curl -H "x-api-key: $KEY" "http://localhost:10400/api/v1/audit/session/session-uuid-aqui"

# Eventos de un usuario específico (últimos 50)
curl -H "x-api-key: $KEY" "http://localhost:10400/api/v1/audit/events?userId=user-uuid&limit=50"

# Health check del módulo de auditoría (requiere x-api-key igual que el resto, salvo AUDIT_API_KEY vacío)
curl -H "x-api-key: $KEY" "http://localhost:10400/api/v1/audit/health"

# Health check global del servicio (sin key, sin prefijo de versión)
curl "http://localhost:10400/health"
```

## Variables de entorno

| Variable | Requerida | Default | Descripción |
|----------|:---------:|---------|-------------|
| `PORT` | — | `10400` | Puerto HTTP |
| `NODE_ENV` | — | `development` | Entorno (`development` / `production`) |
| `AUDIT_API_KEY` | Prod ✓ | — | API key para proteger los endpoints HTTP de consulta. Vacío = permisivo |
| `AUDIT_PAYLOAD_KEY` | Prod ✓ | — | Clave AES-256-GCM de **exactamente 32 bytes UTF-8** para cifrar `payload_encrypted`. Vacío = JSON plano |
| `KAFKA_BROKER` | ✓ | — | Broker(s) Kafka (coma-separados) — este servicio actúa como **consumer** |
| `KAFKA_TOPIC` | — | `platform.logs` | Topic del que consume eventos |
| `DB_HOST` | ✓ | — | Host del SQL Server |
| `DB_PORT` | — | `1433` | Puerto SQL Server |
| `DB_USER` | ✓ | — | Usuario SQL Server |
| `DB_PASS` | ✓ | — | Contraseña SQL Server |
| `DB_NAME` | — | `HCE_AUDIT` | Nombre de la base de datos |
| `DB_INSTANCE` | — | — | Instancia nombrada de SQL Server (vacío si se usa puerto directo) |

## Notas

- Las tablas se crean automáticamente en el primer arranque en `NODE_ENV=development`
- En producción `synchronize` está desactivado — usar migraciones TypeORM
- `payload_encrypted` se cifra con **AES-256-GCM** cuando `AUDIT_PAYLOAD_KEY` está configurado. El formato antes de persistir es `base64(iv).base64(authTag).base64(ciphertext)`, almacenado como `VARBINARY(MAX)` (columna binaria, no texto). Sin key, se almacena JSON plano igualmente convertido a binario (solo desarrollo)
- Generar una key segura: `openssl rand -base64 24 | tr -d '=' | head -c 32`

## Cómo ejecutar

### Local sin Docker

Requiere acceso a SQL Server y a un broker Kafka.

```bash
npm install
# Copiar .env.example a .env y completar DB_HOST, DB_USER, DB_PASS, KAFKA_BROKER
# En producción: establecer AUDIT_API_KEY y AUDIT_PAYLOAD_KEY con valores seguros
npm run start:dev
```

### Local con Docker

Levanta Kafka y el servicio juntos con `docker-compose.dev.yml`:

```bash
docker compose -f docker-compose.dev.yml build
docker compose -f docker-compose.dev.yml up -d

# O build + up en un solo comando:
docker compose -f docker-compose.dev.yml up -d --build

# Para bajar:
docker compose -f docker-compose.dev.yml down
```

Si SQL Server corre en tu máquina, usar `DB_HOST=host.docker.internal` en `.env`. `KAFKA_BROKER` en `.env` puede ser cualquier valor — dentro del contenedor siempre se usa `kafka:9092` (red interna Docker).

### Producción (con Vault)

El `docker-compose.yml` inyecta `VAULT_TOKEN` vía `env_file: .env.docker` (no como variable de entorno
exportada); `entrypoint.sh` usa ese token para leer el resto de los secretos directamente desde Vault
al arrancar. **No se necesita `.env`** con los secretos de la app.

**Requisito:** Vault corriendo en `192.168.42.44:8200` (ver [HCE-vault-config](../HCE-vault-config/README.md)).

#### Paso 1 — Obtener el token

El archivo `HCE-vault-config/.env` tiene la línea:
```
TOKEN_LOGS_SERVICE=hvs.CAESIDsn...
```
Copia ese valor.

#### Paso 2 — Crear `.env.docker` con el token

Este archivo tiene **una sola línea** con el token de bootstrap. No contiene secretos de la app — esos vienen del vault.

**PowerShell (Windows):**
```powershell
"VAULT_TOKEN=hvs.CAESIDsn..." | Out-File -Encoding utf8 .env.docker
```

**Bash / Linux / Mac:**
```bash
echo "VAULT_TOKEN=hvs.CAESIDsn..." > .env.docker
```

> `.env.docker` está en `.gitignore` — nunca se commitea.
> Si el init regenera los tokens, actualizar este archivo con el nuevo valor de `TOKEN_LOGS_SERVICE`.

#### Paso 3 — Levantar

```bash
docker compose down
docker compose build
docker compose up -d
```

Al arrancar, `entrypoint.sh` obtiene `DB_PASS`, `DB_HOST`, `KAFKA_EXTERNAL_HOST`, `AUDIT_API_KEY`,
`AUDIT_PAYLOAD_KEY` y el resto desde Vault (`hce/nestjs/tch-audit-logger`). `KAFKA_BROKER=kafka:9092`
lo fija el `docker-compose.yml` directamente (red interna Docker, no viene de Vault).

> **Nota:** `AUDIT_API_KEY` y `AUDIT_PAYLOAD_KEY` también deben agregarse a Vault (`secret/hce/nestjs/tch-audit-logger`) antes del primer deploy en producción.

Con GitHub Actions el token se pasa como variable de entorno desde GitHub Secrets (`TOKEN_LOGS_SERVICE`).

---

## Scripts disponibles

```bash
npm run start:dev   # desarrollo con hot-reload
npm run build       # compilar TypeScript
npm run start:prod  # ejecutar build
npm run test        # tests unitarios
npm run test:cov    # cobertura
```
