# RC-Final — Local Development Setup Guide

---

## Table of Contents

1. [What is RC-Final?](#what-is-rc-final)
2. [Prerequisites](#prerequisites)
3. [System Requirements](#system-requirements)
4. [Important Concepts](#important-concepts)
5. [Installation Steps](#installation-steps)
   - [Step 1: Prepare the project location](#step-1-prepare-the-project-location)
   - [Step 2: Fix line endings in shell scripts](#step-2-fix-line-endings-in-shell-scripts)
   - [Step 3: Fix PostgreSQL volume in docker-compose.yml](#step-3-fix-postgresql-volume-in-docker-composeyml)
   - [Step 4: Initialize and unseal Vault](#step-4-initialize-and-unseal-vault)
   - [Step 5: Verify all services are running](#step-5-verify-all-services-are-running)
   - [Step 6: Add Keycloak to hosts file](#step-6-add-keycloak-to-hosts-file)
   - [Step 7: Regenerate Keycloak client secret](#step-7-regenerate-keycloak-client-secret)
   - [Step 8: Update KEYCLOAK_SECRET in .env](#step-8-update-keycloak_secret-in-env)
   - [Step 9: Restart registry to pick up new secret](#step-9-restart-registry-to-pick-up-new-secret)
   - [Step 10: Disable SSL requirement in Keycloak](#step-10-disable-ssl-requirement-in-keycloak)
   - [Step 11: Final verification](#step-11-final-verification)
6. [Daily Operations](#daily-operations)
7. [Issues & Solutions](#issues--solutions)
   - [Issue 1: Shell script CRLF line endings](#issue-1-shell-script-crlf-line-endings)
   - [Issue 2: PostgreSQL can't set permissions on db-data folder](#issue-2-postgresql-cant-set-permissions-on-db-data-folder)
   - [Issue 3: Vault-data mount path conflict](#issue-3-vault-data-mount-path-conflict)
   - [Issue 4: Identity service fails with Prisma migration error](#issue-4-identity-service-fails-with-prisma-migration-error)
   - [Issue 5: Registry fails with UnknownHostException: credential-schema](#issue-5-registry-fails-with-unknownhostexception-credential-schema)
   - [Issue 6: Vault sealed after every restart](#issue-6-vault-sealed-after-every-restart)
   - [Issue 7: Docker Compose version warning](#issue-7-docker-compose-version-warning)
8. [Services Reference](#services-reference)
9. [Next Steps After Setup](#next-steps-after-setup)

---

## What is RC-Final?

RC-Final (Registry & Credentialling 2.0) is a **digital credential issuance platform**. You define schemas (templates for certificates), issue verifiable W3C credentials (signed digital certificates), and anyone can verify them cryptographically without calling the issuer.

| Component | Analogy | What it does |
|---|---|---|
| **Registry** | Main office | Processes applications, stores records, manages entities |
| **Keycloak** | Security guard | Handles logins, user roles, and permissions |
| **Identity** | ID card printer | Creates and manages Decentralized Identifiers (DIDs) |
| **Credential Schema** | Form designer | Defines the structure of a certificate |
| **Credential** | Certificate printer | Creates signed, tamper-proof digital certificates |
| **Vault** | Bank safe | Stores cryptographic signing keys securely |
| **PostgreSQL** | Filing cabinet | Stores all data |

Flow: define a schema → submit data → credential signed with Vault keys → anyone can verify.

---

## Prerequisites

| Tool | Why | Check |
|---|---|---|
| **Docker** | Runs each service in a container | `docker -v` |
| **Docker Compose** | Starts all containers together | `docker compose version` |
| **make** | Runs the Vault setup script | `make --version` |
| **sed** | Fixes line endings (Windows/WSL) | `sed --version` |
| **nano** (or vim) | Edit config files | `nano --version` |

If missing: `sudo apt install <tool>` (Ubuntu/WSL). On WSL, ensure Docker Desktop uses WSL 2 backend.

---

## System Requirements

| Resource | Minimum | Why |
|---|---|---|
| **CPU** | 4 cores | 9 services + databases run simultaneously |
| **RAM** | 8 GB system total | Containers use: Yugabyte (~2GB), ES (~1GB), Java (~1-2GB each), Kafka (~512MB) |
| **Disk** | 25 GB free | Docker images (~6GB), data volumes, dependencies |
| **Docker RAM** | 6 GB allocated | Default is 3.8 GB — NOT enough. Settings > Resources > Memory > 6144 MB |

---

## Important Concepts

### 1. WSL filesystem matters

On WSL, keep your project in `/home/...` (ext4), NOT `/mnt/c/...` (NTFS). Docker bind mounts need Linux permissions (`chmod`, `chown`). NTFS doesn't support them — PostgreSQL will crash on startup.

### 2. Vault & Shamir's Secret Sharing

Vault securely stores cryptographic signing keys. On restart, it **seals** (locks) itself — all secrets become inaccessible. To unlock, you need 3 of 5 **unseal keys** generated during `vault operator init`. The keys never change. Save `keys.txt` permanently.

### 3. Keycloak SSL

Keycloak requires HTTPS by default. Registry talks to it over HTTP (`http://keycloak:8080/auth`). We fix this by updating the database: `ssl_required = 'NONE'` for `master` and `sunbird-rc` realms.

### 4. OAuth2 Client Credentials

Registry authenticates to Keycloak using client ID `admin-api` + a secret. The default secret in the imported realm is a placeholder — we regenerate it and save to `.env` as `KEYCLOAK_SECRET`.

### 5. Service dependency chain

```
db → keycloak, vault → identity → credential-schema → credential
db → registry → claim-ms → nginx
```

Services start only after their dependencies pass health checks (~1-2 min).

---

# INSTALLATION STEPS

---

## Step 1: Prepare the project location

**Purpose:** Place the project on a filesystem that works with Docker bind mounts.

**Linux:**
```bash
git clone <repo-url> rc-final
cd rc-final
```

**WSL (must be on ext4, NOT NTFS):**
```bash
cd ~
mkdir -p sunbird
cp -r /mnt/c/.../rc-final ~/sunbird/
cd ~/sunbird/rc-final
```

**Verify:**
```bash
pwd    # should show /home/.../sunbird/rc-final
ls -la # should see docker-compose.yml, .env, setup_vault.sh
```

> See [Concept 1](#1-wsl-filesystem-matters) for why. See [Issue 2](#issue-2-postgresql-cant-set-permissions-on-db-data-folder) if PostgreSQL fails later.


## Step 2: Fix line endings in shell scripts

**Purpose:** Git on Windows adds CRLF (`\r\n`) to scripts. Bash on Linux chokes on `\r`.

```bash
sed -i 's/\r$//' setup_vault.sh
```

**Verify:**
```bash
cat -A setup_vault.sh | head -3   # lines should end with $ only, not ^M$
```

> See [Issue 1](#issue-1-shell-script-crlf-line-endings) if you see `$'\r': command not found` errors.



## Step 3: Fix PostgreSQL volume in docker-compose.yml

**Purpose:** Replace bind mount with a Docker named volume to avoid permission issues.

**Edit `docker-compose.yml`:**

1. In the `db:` service, replace:
   ```yaml
   # Remove this:
   - ./${DB_DIR-db-data}:/var/lib/postgresql/data
   # Add this:
   - db-data:/var/lib/postgresql/data
   ```

2. At the bottom of the file, add:
   ```yaml
   volumes:
     db-data:
   ```

> See [Issue 2](#issue-2-postgresql-cant-set-permissions-on-db-data-folder) if you get `initdb: could not change permissions`.

---

## Step 4: Initialize and unseal Vault

**Purpose:** One-time Vault setup — generates signing keys, unseals Vault, starts all services.

```bash
make compose-init
```

This runs `setup_vault.sh` (init + unseal Vault) then `docker-compose up -d`.

**Verify Vault is unsealed:**
```bash
docker exec rc-final-vault-1 vault status
# Sealed: false  ← what we want
```

**Verify keys were created (SAVE THIS FILE):**
```bash
cat keys.txt   # 5 unseal keys + root token
```

**Verify .env was updated:**
```bash
grep VAULT_TOKEN .env   # should show actual token, not ************
```

> **Issues that can occur here:**
> - `Sealed: true` → [Issue 6: Vault sealed after restart](#issue-6-vault-sealed-after-every-restart)
> - `file exists` error → [Issue 3: Vault-data mount conflict](#issue-3-vault-data-mount-path-conflict)
> - `version is obsolete` warning → [Issue 7: Compose version warning](#issue-7-docker-compose-version-warning) (cosmetic)

---

## Step 5: Verify all services are running

**Purpose:** Confirm all 9 containers are healthy.

```bash
docker-compose ps
```

**Expected — all 9 show `Up (healthy)`:**
```
rc-final-db-1                Up (healthy)
rc-final-vault-1             Up (healthy)
rc-final-keycloak-1          Up (healthy)
rc-final-identity-1          Up (healthy)
rc-final-credential-schema-1 Up (healthy)
rc-final-credential-1        Up (healthy)
rc-final-registry-1          Up (healthy)
rc-final-claim-ms-1          Up (healthy)
rc-final-nginx-1             Up (healthy)
```

**If some are unhealthy**, wait 2 minutes and check again. If still broken, fix in this order:
1. **Identity unhealthy** → [Issue 4: Prisma migration error](#issue-4-identity-service-fails-with-prisma-migration-error)
2. **Registry unhealthy** → [Issue 5: UnknownHostException](#issue-5-registry-fails-with-unknownhostexception-credential-schema)

---

## Step 6: Add Keycloak to hosts file

**Purpose:** Your browser can't resolve Docker's internal hostname `keycloak`. We map it to `127.0.0.1`.

```bash
echo "127.0.0.1   keycloak" | sudo tee -a /etc/hosts
```

**Verify:**
```bash
getent hosts keycloak   # should show 127.0.0.1
```

On Windows (non-WSL): add same line to `C:\Windows\System32\drivers\etc\hosts` as Administrator.

---

## Step 7: Regenerate Keycloak client secret

**Purpose:** The registry needs a fresh secret to authenticate with Keycloak (see [Concept 4](#4-oauth2-client-credentials)).

1. Open browser → `http://localhost:8080/auth/`
2. Login: `admin` / `admin`
3. Top-left dropdown → switch from `master` to **`sunbird-rc`**
4. Left sidebar → **Clients** → click **`admin-api`**
5. **Credentials** tab → **Regenerate Secret** → **copy the new secret**

---

## Step 8: Update KEYCLOAK_SECRET in .env

**Purpose:** Registry reads the secret from `.env` at startup.

```bash
nano .env
```

Find `KEYCLOAK_SECRET=************` and replace `************` with the secret you copied in Step 7.

```
KEYCLOAK_SECRET=WZx8F3pGjKm9Qr2Lv5Nc6Yh7Bs1Ae4Ud   # your actual value
```

---

## Step 9: Restart registry to pick up new secret

**Purpose:** Environment variables are read only at container startup. The running container doesn't see the `.env` change.

```bash
docker restart rc-final-registry-1
sleep 30
docker-compose ps | grep registry
# Should show: rc-final-registry-1  Up (healthy)
```

If it stays unhealthy:
- `UnknownHostException` → [Issue 5](#issue-5-registry-fails-with-unknownhostexception-credential-schema)
- `401 Unauthorized` → secret still wrong, repeat Steps 7-8

---

## Step 10: Disable SSL requirement in Keycloak

**Purpose:** Keycloak blocks HTTP by default. We tell it to allow plain HTTP for local dev (see [Concept 3](#3-keycloak-ssl)).

```bash
# Connect to PostgreSQL inside the db container
docker exec -it rc-final-db-1 psql -U postgres -d registry

# Run these SQL commands:
UPDATE REALM SET ssl_required = 'NONE' WHERE id = 'master';
UPDATE REALM SET ssl_required = 'NONE' WHERE id = 'sunbird-rc';

# Exit:
\q
```

**Restart Keycloak** (it caches settings in memory):
```bash
docker restart rc-final-keycloak-1
sleep 10
docker-compose ps | grep keycloak   # should show Up (healthy)
```

---

## Step 11: Final verification

```bash
# All 9 containers healthy?
docker-compose ps

# Registry health endpoint
curl http://localhost:8081/health

# Registry API responds
curl -v http://localhost:8081/
```

---

# DAILY OPERATIONS

After the initial setup, these are the everyday commands.

---

## Starting everything (after reboot or full stop)

```bash
cd ~/sunbird/rc-final
docker-compose up -d
docker exec rc-final-vault-1 vault operator unseal <Key 1>
docker exec rc-final-vault-1 vault operator unseal <Key 2>
docker exec rc-final-vault-1 vault operator unseal <Key 3>
sleep 120
docker-compose ps
```

> **Speed tip:** Create `unseal.sh` with your 3 keys, then: `docker-compose up -d && ./unseal.sh && sleep 120 && docker-compose ps`

---

## Graceful shutdown

```bash
cd ~/sunbird/rc-final
docker-compose down          # preserves all data
```

To also wipe all data (factory reset):
```bash
docker-compose down -v
rm -rf keys.txt vault-data/
```

---

## Restart a single service

```bash
docker restart rc-final-<service>-1
```

| Command | When |
|---|---|
| `docker restart rc-final-registry-1` | After changing `KEYCLOAK_SECRET` |
| `docker restart rc-final-identity-1` | After fixing Prisma migration |
| `docker restart rc-final-keycloak-1` | After changing SSL / realm settings |

---

## Checking status

```bash
docker-compose ps                          # all healthy?
docker logs rc-final-<service>-1 --tail 30 # check errors
```

---

## What's NOT needed on subsequent starts

| Initial step | Needed again? | Why |
|---|---|---|
| Steps 1-3 (project, line endings, volume) | No | One-time fixes |
| Step 4 (init Vault) | No | Already initialized |
| Step 6 (hosts file) | No | Already in `/etc/hosts` |
| Steps 7-9 (Keycloak secret) | No | Already saved in `.env` |
| Step 10 (disable SSL) | No | Persists in database |
| **Unseal Vault** | **✅ Every restart** | Vault seals itself by design |

---

## If a service won't become healthy

1. `docker-compose ps` — which service?
2. `docker logs rc-final-<service>-1 --tail 30` — what error?
3. Fix by priority:
   - **Vault sealed** → run 3 unseal commands
   - **Identity unhealthy** → [Issue 4](#issue-4-identity-service-fails-with-prisma-migration-error)
   - **Registry unhealthy** → [Issue 5](#issue-5-registry-fails-with-unknownhostexception-credential-schema)

---

# ISSUES & SOLUTIONS

## Issue 1: Shell script CRLF line endings

**Error:**
```
bash setup_vault.sh
$'\r': command not found   (repeated on every line)
```

**Cause:** Git on Windows converts LF (`\n`) to CRLF (`\r\n`). Bash sees `\r` as a command name.

**Fix:**
```bash
sed -i 's/\r$//' setup_vault.sh
```

**Verify:** `cat -A setup_vault.sh | head -3` → lines end with `$` (not `^M$`).

---

## Issue 2: PostgreSQL can't set permissions on db-data folder

**Error:**
```
rc-final-db-1 | chmod: /var/lib/postgresql/data: Operation not permitted
rc-final-db-1 | initdb: error: could not change permissions of directory
```

Container exits immediately with code 1.

**Cause:** The `docker-compose.yml` bind-mounts `./db-data` from the host. On NTFS (or even ext4 in some cases), PostgreSQL can't `chown`/`chmod` its data directory.

**Fix:** Replace the bind mount with a Docker named volume (see [Step 3](#step-3-fix-postgresql-volume-in-docker-composeyml)).

---

## Issue 3: Vault-data mount path conflict

**Error:**
```
Error response from daemon: mkdir /mnt/c/.../vault-data: file exists
```

**Cause:** A leftover `vault-data` directory from a previous failed run.

**Fix:**
```bash
rm -rf vault-data
docker-compose up -d
```

---

## Issue 4: Identity service fails with Prisma migration error

**Error:** `docker logs rc-final-identity-1 --tail 20`
```
Error: P3005
The database schema is not empty.
```

Identity shows `Exited (1)` or stays `Created`.

**Cause:** Prisma tries to run migrations but finds the database already has tables (created by another service in a race condition). It refuses to proceed.

**Fix:**
```bash
# Mark the initial migration as already applied
docker exec -it rc-final-identity-1 sh -c "npx prisma migrate resolve --applied 0_init"

# Restart identity
docker restart rc-final-identity-1
sleep 20
docker-compose ps | grep identity   # should show Up (healthy)
```

---

## Issue 5: Registry fails with UnknownHostException: credential-schema

**Error:** `docker logs rc-final-registry-1 --tail 20`
```
java.net.UnknownHostException: credential-schema
```

Registry shows `Up (unhealthy)` and never becomes healthy.

**Cause:** The dependency chain is `identity → credential-schema → registry`. If identity failed (Issue 4), credential-schema never starts, and registry can't reach it.

**Fix:** Fix identity first (Issue 4 fix above), then:
```bash
docker restart rc-final-registry-1
sleep 30
docker-compose ps | grep registry   # should show Up (healthy)
```

---

## Issue 6: Vault sealed after every restart

**Error:** `docker exec rc-final-vault-1 vault status` shows `Sealed: true`.

**Cause:** This is **by design** — Vault locks itself on restart as a security measure. It needs 3 of 5 unseal keys to decrypt its data.

**Fix:**
```bash
# Run 3 unseal commands with keys from keys.txt
docker exec rc-final-vault-1 vault operator unseal <Key 1>
docker exec rc-final-vault-1 vault operator unseal <Key 2>
docker exec rc-final-vault-1 vault operator unseal <Key 3>

# Verify
docker exec rc-final-vault-1 vault status   # Sealed: false
```

---

## Issue 7: Docker Compose version warning

**Warning:**
```
WARN[0000] the attribute `version` is obsolete, it will be ignored
```

**Cause:** The `docker-compose.yml` starts with `version: '2.4'`. New Docker Compose (v2+) ignores this field.

**Fix:** Cosmetic only. Remove `version: '2.4'` from the first line of `docker-compose.yml` if the warning bothers you.

---

# SERVICES REFERENCE

| Service | Port | Purpose | Depends On |
|---|---|---|---|
| **db** (PostgreSQL 15) | 5432 | Single database for all services | — |
| **vault** (HashiCorp Vault 1.13.3) | 8200 | Secure key storage (Shamir secret sharing) | db |
| **keycloak** (Sunbird RC Keycloak) | 8080, 9990 | OIDC authentication, user roles & permissions | db |
| **identity** (DID Service) | 3332 | Creates/resolves Decentralized Identifiers (Ed25519) | db, vault |
| **credential-schema** | 3333 | Manages JSON-LD credential schema definitions | db, identity |
| **credential** | 3000 | Issues W3C Verifiable Credentials signed by Vault | db, identity, credential-schema |
| **registry** (Sunbird RC Core) | 8081 | Main REST API — entities, schemas, credentials | db, keycloak |
| **claim-ms** | 8082 | Claims and disputes against entities/credentials | db, registry |
| **nginx** | 80, 443 | Reverse proxy routing external traffic | registry, keycloak, claim-ms |

---

# NEXT STEPS AFTER SETUP

1. **Explore schemas** — See examples in `schemas/` folder (`Student.json`, `Teacher.json`, etc.)
2. **Create entities via API** — POST data to the registry using schema definitions
3. **Issue credentials** — Convert registry entities into signed verifiable credentials
4. **Verify credentials** — Check cryptographic validity
5. **Define custom schemas** — Create your own credential templates
6. **Postman collection** — Import from the link in `README.md` to test endpoints

---

*End of document*
