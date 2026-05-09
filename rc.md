# RC-Final — Registry & Credentialling 2.0

## 1. Overview

**RC-Final** is a **digital credential issuance platform**. It defines schemas (templates for certificates), issues verifiable W3C credentials (signed digital certificates), and allows anyone to verify them cryptographically without calling back to the issuer.

### Component Analogy

| Component | Analogy | What it does |
|---|---|---|
| **Registry** | Main office | Processes applications, stores records, manages entities |
| **Keycloak** | Security guard | Handles logins, user roles, and permissions |
| **Identity** | ID card printer | Creates and manages Decentralized Identifiers (DIDs) |
| **Credential Schema** | Form designer | Defines the structure of a certificate |
| **Credential** | Certificate printer | Creates signed, tamper-proof digital certificates |
| **Vault** | Bank safe | Stores cryptographic signing keys securely |
| **PostgreSQL** | Filing cabinet | Stores all data |

### Flow

```
Define schema → Submit entity data → Issue credential → Anyone can verify cryptographically
```

---

## 2. Project Structure

```
rc-final/
├── .env                          # Environment variables (local configuration)
├── .env.example                  # Environment variable template
├── docker-compose.yml            # All service definitions (403 lines)
├── Makefile                      # Automation (make compose-init)
├── README.md                     # Setup guide
├── setup.md                      # Additional setup notes
├── setup_vault.sh                # Vault initialization script
├── vault.json                    # Vault server configuration
├── keys.txt                      # Vault unseal keys + root token (auto-generated)
├── imports/                      # Keycloak realm import
│   ├── realm-export.json
│   └── nginx.conf
├── schemas/                      # JSON Schema definitions for entities
│   ├── ConservationVolunteer.json
│   ├── Official.json
│   ├── Insurance.json
│   └── ...
└── rc-demo/                      # Demo frontend (linked from nginx)
```

---

## 3. Services & Tech Stack

### 3.1 Active Services (9 containers)

| Service | Image | Language/Runtime | Port | Purpose |
|---|---|---|---|---|
| **db** | `postgres:15-alpine` | PostgreSQL 15 | 5432 | Single shared database for all services |
| **vault** | `vault:1.13.3` (HashiCorp) | Go | 8200 | Secure cryptographic key storage (Shamir secret sharing) |
| **keycloak** | `ghcr.io/sunbird-rc/sunbird-rc-keycloak:latest` | Java (JBoss/WildFly) | 8080, 9990 | OIDC authentication, SSO, realm management (`sunbird-rc`) |
| **registry** | `ghcr.io/sunbird-rc/sunbird-rc-core:v2.0.0-rc3` | Java (Spring Boot) | 8081 | Main REST API — entity CRUD, schema management, credential orchestration |
| **identity** | `ghcr.io/sunbird-rc/sunbird-rc-identity-service:v2.0.0-rc3` | Node.js (TypeScript) | 3332 | DID (Decentralized Identifier) creation/resolution (Ed25519 keys) |
| **credential-schema** | `ghcr.io/sunbird-rc/sunbird-rc-credential-schema:v2.0.0-rc3` | Node.js (TypeScript) | 3333 | JSON-LD credential schema definition management |
| **credential** | `ghcr.io/sunbird-rc/sunbird-rc-credentials-service:v2.0.1` | Node.js (TypeScript) | 3000 | W3C Verifiable Credential signing, issuance, and verification |
| **claim-ms** | `ghcr.io/sunbird-rc/sunbird-rc-claim-ms:v2.0.0-rc3` | Java (Spring Boot) | 8082 | Claims and disputes management against entities/credentials |
| **nginx** | `nginx:latest` | C / Nginx | 80, 443 | Reverse proxy — routes external traffic to registry, keycloak, claim-ms |

### 3.2 Disabled/Optional Services (commented out)

| Service | Image | Port | Purpose |
|---|---|---|---|
| **file-storage** | `quay.io/minio/minio` | 9000, 9001 | S3-compatible file storage for credential artifacts |
| **kafka** | `confluentinc/cp-kafka:latest` | 9092 | Event-driven workflows |
| **zookeeper** | `confluentinc/cp-zookeeper:latest` | 2181 | Kafka coordination |
| **redis** | `redis:latest` | 6379 | Caching |
| **encryption-service** | `ghcr.io/sunbird-rc/encryption-service` | 8013 | Field-level encryption |
| **id-gen-service** | `ghcr.io/sunbird-rc/id-gen-service` | 8088 | ID generation |
| **notification-ms** | `ghcr.io/sunbird-rc/sunbird-rc-notification-service` | 8765 | Multi-channel notifications |
| **context-proxy-service** | `ghcr.io/sunbird-rc/sunbird-rc-context-proxy-service` | 4400 | JSON-LD context proxy |

---

## 4. Architecture

### 4.1 Network Topology

```
                          ┌─────────────┐
                          │  nginx:80   │
                          │  nginx:443  │
                          └──────┬──────┘
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                   ▼
       ┌─────────────┐   ┌──────────────┐   ┌──────────────┐
       │  registry   │   │   keycloak   │   │   claim-ms   │
       │  :8081      │   │   :8080      │   │   :8082      │
       └──────┬──────┘   └──────┬───────┘   └──────────────┘
              │                 │
              │    ┌────────────┼────────────┐
              │    ▼            ▼            ▼
              │ ┌────────┐ ┌────────┐ ┌────────────┐
              │ │identity│ │cred-   │ │ credential │
              │ │:3332   │ │schema  │ │ :3000      │
              │ │(DID)   │ │:3333   │ │(VC issue)  │
              │ └───┬────┘ └────────┘ └────────────┘
              │     │
              │     ▼
              │ ┌───────┐
              │ │ vault │
              │ │:8200  │
              │ └───────┘
              │
              ▼
       ┌──────────────┐
       │  db (PG 15)  │
       │  :5432       │
       └──────────────┘
```

### 4.2 Dependency Chain

```
db → keycloak, vault → identity → credential-schema → credential
db → registry → claim-ms → nginx
```

Services start only after their dependencies pass health checks (~1-2 min).

---

## 5. Technology Stack Summary

| Category | Technology |
|---|---|
| **Database** | PostgreSQL 15 (single shared instance, all services) |
| **Authentication** | Keycloak (OIDC), JWT bearer tokens, realm: `sunbird-rc` |
| **Key Management** | HashiCorp Vault 1.13.3 (Shamir Secret Sharing, 5 keys, 3 required to unseal) |
| **Identity** | DIDs (Decentralized Identifiers) on Ed25519 curve |
| **Credentials** | W3C Verifiable Credentials (JSON-LD format) |
| **Registry API** | Java 11, Spring Boot (`sunbird-rc-core`) |
| **Identity Service** | Node.js, TypeScript |
| **Credential Schema** | Node.js, TypeScript |
| **Credential Service** | Node.js, TypeScript |
| **Claims Service** | Java 11, Spring Boot |
| **Reverse Proxy** | Nginx (latest) |
| **Container Runtime** | Docker, Docker Compose (version 2.4) |
| **Demo Frontend** | Vanilla HTML/CSS/JS (served by nginx at `/demo`) |

---

## 6. Service Details

### 6.1 db — PostgreSQL 15

**Image:** `postgres:15-alpine`
**Port:** 5432
**Database:** `registry`
**User:** `postgres` / `postgres`

All services share a single PostgreSQL database instance. Each service manages its own schema and tables.

**Health check:**
```bash
pg_isready -U postgres
```

### 6.2 vault — HashiCorp Vault

**Image:** `vault:1.13.3`
**Port:** 8200
**Config:** `vault.json` (mounted at `/vault/config/vault.json`)
**Storage:** File backend at `/vault/file` (persisted to `./vault-data/`)

**Key Concepts:**
- **Seal/Unseal:** Vault seals itself on restart. Requires 3 of 5 unseal keys to unlock.
- **Shamir Secret Sharing:** The master key is split into 5 shards. Any 3 are needed to reconstruct it.
- **Keys file:** `keys.txt` is auto-generated during `make compose-init`. Save it permanently.

**Environment variables:**
```
VAULT_ADDR=http://0.0.0.0:8200
VAULT_API_ADDR=http://0.0.0.0:8200
VAULT_ADDRESS=http://0.0.0.0:8200
```

**Capabilities:** `IPC_LOCK` (required for Vault's secure memory locking)

**Health check:**
```bash
wget --spider http://127.0.0.1:8200/v1/sys/health
```

### 6.3 keycloak — Sunbird RC Keycloak

**Image:** `ghcr.io/sunbird-rc/sunbird-rc-keycloak:latest`
**Ports:** 8080 (HTTP), 9990 (management)
**Database:** PostgreSQL (`registry` database)
**Realm:** `sunbird-rc` (imported from `imports/realm-export.json`)

**Key Configuration:**
- Admin user: `admin` / `admin`
- Client `admin-api`: Used by registry for OIDC authentication
- Client `registry-frontend`: Public client for frontend flows
- SSL: Disabled for local dev (`ssl_required = NONE` in PostgreSQL)

**Environment variables:**
```
DB_VENDOR=postgres
DB_ADDR=db
DB_PORT=5432
DB_DATABASE=registry
DB_USER=postgres
DB_PASSWORD=postgres
KEYCLOAK_USER=admin
KEYCLOAK_PASSWORD=admin
KEYCLOAK_IMPORT=/opt/jboss/keycloak/imports/realm-export.json
```

**Health check:**
```bash
curl -f http://localhost:9990/
```

### 6.4 registry — Sunbird RC Core

**Image:** `ghcr.io/sunbird-rc/sunbird-rc-core:v2.0.0-rc3`
**Port:** 8081
**Language:** Java (Spring Boot)

The central API gateway for the platform. Handles entity registration, schema management, credential issuance orchestration, and connects to all upstream services.

**Key capabilities:**
- Entity CRUD (create, read, update, delete) against PostgreSQL
- Schema definitions (JSON Schema draft-07)
- Credential issuance flow (orchestrates identity → schema → credential services)
- Role-based access via Keycloak
- Registry API base path: `/registry/api/v1/`
- Swagger docs: `http://localhost:8081/api/docs/swagger.json`

**Environment variables (key):**
```
connectionInfo_uri=jdbc:postgresql://db:5432/registry
sunbird_sso_url=http://keycloak:8080/auth
sunbird_sso_admin_client_secret=${KEYCLOAK_SECRET}
signature_v2_issue_url=http://credential:3000/credentials/issue
did_generate_url=http://identity:3332/did/generate
```

**Health check:**
```bash
wget -nv -t1 --spider http://localhost:8081/health
```

### 6.5 identity — DID Service

**Image:** `ghcr.io/sunbird-rc/sunbird-rc-identity-service:v2.0.0-rc3`
**Port:** 3332
**Language:** Node.js (TypeScript)

Creates and resolves Decentralized Identifiers (DIDs). Uses Ed25519 keys stored in Vault.

**API endpoints:**
- `GET /health` — Health check
- `POST /did/generate` — Generate a new DID
- `GET /did/resolve/{id}` — Resolve a DID to its DID Document

**Environment variables:**
```
DATABASE_URL=postgres://postgres:postgres@db:5432/registry
VAULT_ADDR=http://vault:8200
VAULT_TOKEN=${VAULT_TOKEN}
SIGNING_ALGORITHM=Ed25519
```

**Health check:**
```bash
curl -f http://localhost:3332/health
```

### 6.6 credential-schema — Schema Service

**Image:** `ghcr.io/sunbird-rc/sunbird-rc-credential-schema:v2.0.0-rc3`
**Port:** 3333
**Language:** Node.js (TypeScript)

Manages JSON-LD credential schema definitions. Templates that define the structure of verifiable credentials.

**API endpoints:**
- `GET /health` — Health check
- `POST /credential-schema` — Create a new schema
- `GET /credential-schema/{id}/{version}` — Get schema by ID and version
- `GET /credential-schema?tags={tags}` — Search schemas by tags

**Health check:**
```bash
curl -f http://localhost:3333/health
```

### 6.7 credential — Credentials Service

**Image:** `ghcr.io/sunbird-rc/sunbird-rc-credentials-service:v2.0.1`
**Port:** 3000
**Language:** Node.js (TypeScript)

Handles the core credential lifecycle: issue, verify, revoke. Signs W3C Verifiable Credentials using keys stored in Vault (via identity service DIDs).

**API endpoints:**
- `GET /health` — Health check
- `POST /credentials/issue` — Issue a signed credential
- `GET /credentials/{id}` — Get credential by ID
- `DELETE /credentials/{id}` — Delete credential
- `POST /credentials/{id}/verify` — Verify a specific credential
- `POST /credentials/verify` — Verify any credential
- `GET /credentials/revocation-list` — Get revocation list

**Health check:**
```bash
curl -f http://localhost:3000/health
```

### 6.8 claim-ms — Claims Service

**Image:** `ghcr.io/sunbird-rc/sunbird-rc-claim-ms:v2.0.0-rc3`
**Port:** 8082
**Language:** Java (Spring Boot)

Manages claims and disputes against entities and credentials. Allows users to challenge or request changes to issued credentials.

**Environment variables:**
```
connectionInfo_uri=jdbc:postgresql://db:5432/registry
sunbirdrc_url=http://registry:8081
```

**Health check:**
```bash
wget -nv -t1 --spider http://localhost:8082/health
```

### 6.9 nginx — Reverse Proxy

**Image:** `nginx:latest`
**Ports:** 80, 443

Routes external HTTP traffic to internal services. Serves the rc-demo frontend at `/demo`.

**Nginx config:** `imports/nginx.conf` (mounted at `/etc/nginx/conf.d/default.conf`)

**Routing:**
| External Path | Internal Target |
|---|---|
| `/auth/*` | `keycloak:8080` |
| `/registry/*` | `registry:8081` |
| `/demo/*` | Static files from `../rc-demo` |
| `/*` (other) | `nginx` default handling |

**Health check:**
```bash
curl -f http://localhost:80/
```

---

## 7. Environment Variables

### 7.1 Required Variables

| Variable | Default | Description |
|---|---|---|
| `RELEASE_VERSION` | `v2.0.0-rc3` | Version tag for Sunbird RC images |
| `KEYCLOAK_SECRET` | `************` | OAuth2 client secret for `admin-api` client (must be regenerated in Keycloak UI) |
| `VAULT_TOKEN` | `************` | Vault root token (auto-generated by `setup_vault.sh`) |

### 7.2 Optional Variables

| Variable | Default | Description |
|---|---|---|
| `VAULT_ADDR` | `http://0.0.0.0:8200` | Vault server address |
| `VAULT_API_ADDR` | `http://0.0.0.0:8200` | Vault API address |
| `VAULT_BASE_URL` | `http://vault:8200/v1` | Vault API base URL |
| `VAULT_ROOT_PATH` | `secret` | Vault secrets engine root path |
| `VAULT_TIMEOUT` | `5000` | Vault request timeout (ms) |
| `VAULT_PROXY` | `""` | Vault proxy URL |
| `SIGNING_ALGORITHM` | `Ed25519` | Cryptographic signing algorithm |
| `JWKS_URI` | `""` | JWKS URI for key resolution |
| `ENABLE_AUTH` | `false` | Enable authentication for schema/credential services |
| `WEB_DID_BASE_URL` | `""` | Base URL for web DIDs |
| `IDENTITY_BASE_URL` | `http://identity:3332` | Identity service base URL |
| `SCHEMA_BASE_URL` | `http://credential-schema:3333` | Schema service base URL |
| `CREDENTIAL_SERVICE_BASE_URL` | `http://credential:3000` | Credential service base URL |
| `QR_TYPE` | `""` | QR code type for credentials |
| `KEYCLOAK_REALM` | `sunbird-rc` | Keycloak realm name |
| `KEYCLOAK_ADMIN_CLIENT_ID` | `admin-api` | Keycloak admin client ID |
| `KEYCLOAK_CLIENT_ID` | `registry-frontend` | Keycloak registry client ID |
| `KEYCLOAK_ADMIN_USER` | `admin` | Keycloak admin username |
| `KEYCLOAK_ADMIN_PASSWORD` | `admin` | Keycloak admin password |
| `SCHEMA_DIR` | `java/registry/src/main/resources/public/_schemas` | Schema files directory |
| `VIEW_DIR` | `java/registry/src/main/resources/views` | View templates directory |
| `KEYCLOAK_IMPORT_DIR` | `imports` | Keycloak realm import directory |
| `ENCRYPTION_ENABLED` | `false` | Enable field-level encryption |
| `EVENT_ENABLED` | `false` | Enable event publishing |
| `DID_ENABLED` | `false` | Enable DID generation |
| `SIGNATURE_ENABLED` | `false` | Enable credential signing |
| `CLAIMS_ENABLED` | `false` | Enable claims service |
| `CERTIFICATE_ENABLED` | `false` | Enable certificate PDF generation |
| `IDGEN_ENABLED` | `false` | Enable ID generation |
| `FILESSTORAGE_ENABLED` | `false` | Enable file storage |
| `AUTHENTICATION_ENABLED` | `false` | Enable API authentication |
| `ASYNC_ENABLED` | `false` | Enable async processing |
| `NOTIFICATION_ASYNC_ENABLED` | `false` | Enable async notifications |

---

## 8. Setup Steps

### 8.1 One-Time Setup

```bash
# 1. Fix line endings if on Windows/WSL
sed -i 's/\r$//' setup_vault.sh

# 2. Initialize Vault and start all services
make compose-init

# 3. Add Keycloak to hosts file
echo "127.0.0.1   keycloak" | sudo tee -a /etc/hosts

# 4. Regen Keycloak client secret
# Open http://localhost:8080/auth → admin/admin
# Clients → admin-api → Credentials → Regenerate Secret
# Copy secret to .env: KEYCLOAK_SECRET=<new-secret>

# 5. Restart registry with new secret
docker restart rc-final-registry-1

# 6. Disable SSL requirement in Keycloak
docker exec -it rc-final-db-1 psql -U postgres -d registry -c \
  "UPDATE REALM SET ssl_required = 'NONE' WHERE id = 'master';"
docker exec -it rc-final-db-1 psql -U postgres -d registry -c \
  "UPDATE REALM SET ssl_required = 'NONE' WHERE id = 'sunbird-rc';"
docker restart rc-final-keycloak-1
```

### 8.2 Daily Operations

```bash
# Start everything
docker compose up -d
docker exec rc-final-vault-1 vault operator unseal <Key1>
docker exec rc-final-vault-1 vault operator unseal <Key2>
docker exec rc-final-vault-1 vault operator unseal <Key3>

# Check status
docker compose ps

# View logs
docker logs rc-final-registry-1 --tail 30

# Restart a single service
docker restart rc-final-registry-1

# Stop (preserves data)
docker compose down

# Reset all data
docker compose down -v
rm -rf keys.txt vault-data/
```

---

## 9. API Reference

### 9.1 Registry API (base: `http://localhost:8081/registry/api/v1/`)

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/Schema` | Admin | Create a schema definition |
| `GET` | `/Schema?name={name}` | Public/Admin | Search schemas |
| `GET` | `/Schema/{id}` | Public/Admin | Get schema by ID |
| `POST` | `/{schemaName}` | Public/Admin | Create entity |
| `GET` | `/{schemaName}` | Public/Admin | List entities |
| `GET` | `/{schemaName}/{osid}` | Public/Admin | Read entity |
| `PATCH` | `/{schemaName}/{osid}` | Admin | Update entity |
| `DELETE` | `/{schemaName}/{osid}` | Admin | Delete entity |
| `POST` | `/utils/sign` | Admin | Issue a signed credential |
| `POST` | `/verify` | Public | Verify a credential |
| `GET` | `/health` | None | Health check |

### 9.2 Keycloak (base: `http://localhost:8080/auth`)

| Path | Description |
|---|---|
| `/realms/sunbird-rc/protocol/openid-connect/token` | OAuth2 token endpoint |
| `/realms/sunbird-rc/protocol/openid-connect/auth` | OAuth2 authorization endpoint |
| `/realms/sunbird-rc/protocol/openid-connect/logout` | OIDC logout |

### 9.3 Credential Schema (base: `http://localhost:3333`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/credential-schema` | Create schema |
| `GET` | `/credential-schema/{id}/{version}` | Get schema |
| `GET` | `/credential-schema?tags={tags}` | Search schemas |

### 9.4 Credential Service (base: `http://localhost:3000`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/credentials/issue` | Issue credential |
| `GET` | `/credentials/{id}` | Get credential |
| `DELETE` | `/credentials/{id}` | Delete credential |
| `POST` | `/credentials/{id}/verify` | Verify credential |
| `POST` | `/credentials/verify` | Verify any credential |
| `GET` | `/credentials/revocation-list` | Get revocation list |

### 9.5 Identity Service (base: `http://localhost:3332`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/did/generate` | Generate new DID |
| `GET` | `/did/resolve/{id}` | Resolve DID document |

---

## 10. Demo Frontend (rc-demo)

The `rc-demo/` directory contains a demo frontend served by nginx at `http://localhost/demo/`.

### Pages

| Page | URL | Purpose |
|---|---|---|
| Home | `/demo/index.html` | Admin setup, schema creation |
| Register | `/demo/register.html` | Register a volunteer entity |
| Directory | `/demo/directory.html` | Browse all registered entities |
| Issue | `/demo/issue.html` | Issue a credential for an entity |
| Verify | `/demo/verify.html` | Paste and verify a credential JSON |

### Architecture

The demo uses vanilla JavaScript (`app.js`) that makes API calls through nginx:

```javascript
// Auth
POST /auth/realms/sunbird-rc/protocol/openid-connect/token

// Schema
POST /registry/api/v1/Schema

// Entities
POST /registry/api/v1/{schemaName}
GET  /registry/api/v1/{schemaName}

// Credentials
POST /registry/utils/sign
POST /registry/api/v1/verify
```

### Running

Serve static files and access via nginx:
```bash
cd rc-demo
python -m http.server 3000
# Or: npx serve .
```

Access at: `http://localhost/demo/` (via nginx proxy) or `http://localhost:3000` (direct)

---

## 11. Common Issues & Solutions

| Issue | Symptom | Fix |
|---|---|---|
| **Vault sealed** | `Sealed: true` | Run 3 unseal commands with keys from `keys.txt` |
| **CRLF line endings** | `$'\r': command not found` | `sed -i 's/\r$//' setup_vault.sh` |
| **PostgreSQL permission error** | `initdb: could not change permissions` | Replace bind mount with Docker named volume |
| **Vault-data mount conflict** | `mkdir: file exists` | `rm -rf vault-data` |
| **Prisma migration error** | `P3005: schema not empty` | `npx prisma migrate resolve --applied 0_init` |
| **Registry UnknownHostException** | `UnknownHostException: credential-schema` | Fix identity service first, then restart registry |
| **Keycloak SSL block** | Registry can't reach Keycloak | Set `ssl_required = 'NONE'` in database |

---

## 12. Security Notes

- **Vault keys** in `keys.txt` are irrecoverable if lost. Back them up.
- **Keycloak admin credentials** default to `admin/admin`. Change in production.
- **Registry API** authentication can be toggled via `AUTHENTICATION_ENABLED` env var.
- **HTTPS** is not configured for local dev. In production, configure SSL in nginx and Keycloak.
- All inter-service communication is over HTTP inside the Docker network (not exposed externally).
