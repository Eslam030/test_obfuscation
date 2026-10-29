# app_server — deployment & usage

This folder is the **client deliverable**: a single compiled binary, nothing else.

| File | What it is |
|---|---|
| `app_server` | The application — Nuitka-compiled, stripped, no source, no keys inside. Runs inside a Linux container. |
| `.env.example` | Every runtime variable the binary needs. Copy to `.env` and fill in. |

The readable source and the build pipeline live elsewhere (the build project); they are **not** shipped here.

---

## 1. What it needs to run

The binary ships with **no secrets and no config baked in**. All of it is injected at runtime:

- A reachable **PostgreSQL** database.
- **Keys** — `CODE_DECRYPT_KEY` (decrypts the licensing module), `DJANGO_SECRET_KEY`, `FIELD_ENCRYPTION_KEY`.
- **Connection / server** vars — `DB_*`, `BIND_*`, optionally `MIGRATE_ON_START`.

See `.env.example` for the full list and notes. Keys are read **once** at startup, then the source file is shredded and the vars are scrubbed from the process environment.

---

## 2. Run it (Docker)

### 2a. Create the database container

```bash
docker network create obfnet 2>/dev/null; \
docker run -d --name wh_test_obs_pg --network obfnet \
  -e POSTGRES_DB=test_obs \
  -e POSTGRES_USER=obs_user \
  -e POSTGRES_PASSWORD=obs_pass \
  -p 5433:5432 \
  postgres:16
```

Empty Postgres with just the `test_obs` database (no tables until the app migrates).

### 2b. Build the image and run the app

The image wraps this exact binary in `debian-slim` + `tini`. Build it from the project that has the `Dockerfile` (the build side), then:

```bash
# keys staged in host RAM, mounted read-once, NOT passed via -e
mkdir -p /dev/shm/obf_keys
{ echo "CODE_DECRYPT_KEY=<your pinned key>"
  echo "DJANGO_SECRET_KEY=$(python3 -c 'import secrets;print(secrets.token_urlsafe(40))')"
  echo "FIELD_ENCRYPTION_KEY=$(python3 -c 'from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())')"
} > /dev/shm/obf_keys/app.env

docker run -d --name obf_app --network obfnet \
  -v /dev/shm/obf_keys:/run/keys -e KEYS_FILE=/run/keys/app.env \
  -e DB_HOST=wh_test_obs_pg -e DB_PORT=5432 -e DB_NAME=test_obs \
  -e DB_USER=obs_user -e DB_PASSWORD=obs_pass \
  -e MIGRATE_ON_START=1 -p 8096:8000 test_obf:exe

rm -rf /dev/shm/obf_keys              # wipe host-side key staging

curl -s localhost:8096/healthz
```

Set `MIGRATE_ON_START=1` only on the **first** run against an empty database; drop it afterward so startup is fast and idempotent.

> **Run it in the container, not on the host.** Running `./app_server` directly on a machine that already has `libpq`/`libssl` (e.g. an active venv) can segfault from library conflicts. The minimal container has no competing libraries.

---

## 3. Endpoints

Base URL below assumes the `-p 8096:8000` mapping above → `http://localhost:8096`.

| Method | Path | Body (JSON) | Purpose | Key needed |
|---|---|---|---|---|
| GET | `/healthz` | — | Liveness + DB reachability | no |
| GET | `/api/echo` | — (query params) | Echoes method + query params | no |
| GET | `/api/config` | — | Effective runtime config (no secret **values**, only booleans) | no |
| GET | `/api/widgets` | — | List widgets | no |
| POST | `/api/widgets` | `{"name","sku","quantity"}` | Create a widget → `201 {id, sku}` | no |
| POST | `/api/quote` | `{"severity","exposure","asset_value","units"}` | Risk score + price (compiled business logic) | no |
| POST | `/api/license` | `{"tier","feature"}` | License tier quota + feature gate (CodeEnigma module) | **yes** |
| GET | `/api/catalog/products` | — | List products | no |
| POST | `/api/catalog/products` | `{"name","sku","price"}` | Create a product → `201 {id, sku}` | no |
| GET | `/api/audit/entries` | — | List audit entries | no |
| POST | `/api/audit/entries` | `{"action","product_id"}` | Create an audit entry → `201 {id, action}` | no |

`/api/license` returns **503** (`licensing unavailable`) if `CODE_DECRYPT_KEY` is absent — the rest of the app works without it.

### Examples

```bash
BASE=http://localhost:8096

curl -s $BASE/healthz
# {"status": "ok", "database": true}

curl -s "$BASE/api/echo?hello=world"
# {"method": "GET", "params": {"hello": "world"}}

curl -s $BASE/api/config

# widgets
curl -s -X POST $BASE/api/widgets -H 'Content-Type: application/json' \
  -d '{"name":"Gadget","sku":"W-1","quantity":3}'
curl -s $BASE/api/widgets

# business logic (no key)
curl -s -X POST $BASE/api/quote -H 'Content-Type: application/json' \
  -d '{"severity":7,"exposure":4,"asset_value":1000,"units":5}'

# licensing (needs CODE_DECRYPT_KEY)
curl -s -X POST $BASE/api/license -H 'Content-Type: application/json' \
  -d '{"tier":"enterprise","feature":"api_access"}'

# catalog + audit
curl -s -X POST $BASE/api/catalog/products -H 'Content-Type: application/json' \
  -d '{"name":"Pro Plan","sku":"C-1","price":"49.99"}'
curl -s -X POST $BASE/api/audit/entries -H 'Content-Type: application/json' \
  -d '{"action":"created","product_id":1}'
```

---

## 4. Verify the key handling

After startup, the key is gone from the container — confirm it:

```bash
docker exec obf_app sh -c 'cat /proc/1/environ | tr "\0" "\n" | grep CODE_DECRYPT || echo "no key in env"'
docker exec obf_app ls /run/keys    # the key file is shredded + removed
```

---

## 5. Stop / reset

```bash
docker rm -f obf_app              # stop the app
docker rm -f wh_test_obs_pg       # drop the database (data is lost)
```
