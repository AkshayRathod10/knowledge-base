# Accessing Postgres on EC2 (UAT/Prod) — Notes

Context: postgres runs inside docker compose on a shared EC2 box (both UAT and
Prod stacks live on the same instance). Goal: connect from a local DB client
(DBeaver) without exposing the DB port to the public internet.

## The core decision: SSH tunnel, not an open SG port

Two ways to reach a DB port on a remote box:

- **Open the port in the Security Group** — SG allows inbound 5432/5433 from
  your IP, DB client connects directly to `<ec2-ip>:5432`. Works, but the DB
  port is now network-reachable (gated only by SG + your IP staying static).
- **SSH tunnel** — DB port stays bound to `127.0.0.1` on the EC2 host (never
  touches the network interface SG polices). Client connects to
  `localhost:5432` on your machine; DBeaver/ssh forwards that through the
  existing SSH session (port 22, already open) to the EC2 host's own
  loopback. No SG change needed at all, since SG rules don't apply to
  loopback traffic.

We used the SSH tunnel approach. Only requirement: port 22 already open
(true — that's how you SSH in), and postgres listening on at least
`127.0.0.1:<port>` on the host.

## docker-compose: one file, env-var-driven port

`docker-compose.prod.yml` is shared by both UAT and Prod (UAT deploys with
the same `prod.yml`, different `.env` file — no separate `docker-compose.uat.yml`
needed). Ports differ per environment via a `DB_PORT` variable:

```yaml
services:
  db:
    ports:
      - "127.0.0.1:${DB_PORT:-5432}:5432"
```

- Prod: leave `DB_PORT` unset in its env file → defaults to `5432`
- UAT: `DB_PORT=5433` in `.env.uat`

**Gotcha:** `${DB_PORT}` substitution here is resolved by **docker compose
itself** at parse time, from the shell env or whatever file is passed via
`--env-file`. This is a *different* mechanism from `env_file:` inside a
service block (which only injects vars into that container's runtime env).
Passing `--env-file .env.uat` is what makes `DB_PORT` (and `DB_NAME`,
`DB_USER`, `DB_PASSWORD`) resolve correctly in the compose file itself.

## Commands used (UAT, `/opt/scorecard-uat`)

```bash
# pull latest compose file changes
git pull

# confirm the ports: line landed
grep -A2 "db:" docker-compose.prod.yml

# edit env file — use Ctrl+O (save) then Ctrl+X (exit) in nano, NOT Ctrl+C
# (Ctrl+C aborts without saving — bit us once)
nano .env.uat
# DB_PORT=5433

# recreate just the db container with new config
docker compose --env-file .env.uat -f docker-compose.yml -f docker-compose.prod.yml up -d db
# Watch for "Recreated" in the output — "Running" means compose saw no
# config diff and did nothing (usually means the file on disk wasn't
# actually updated yet, e.g. forgot to git pull)

# confirm postgres is listening on the host now
sudo ss -tlnp | grep -E '5432|5433'
# LISTEN 127.0.0.1:5433  ->  confirms bound to loopback, ready for tunnel
```

## Running migrations after a deploy

`backend/entrypoint.sh` already runs `migrate --noinput` automatically every
time the `web` container starts, before gunicorn starts. Since prod/uat
don't bind-mount source (only dev's `docker-compose.override.yml` does),
a code change needs an image rebuild first, not just a restart:

```bash
git pull
docker compose --env-file .env.uat -f docker-compose.yml -f docker-compose.prod.yml build web worker beat
docker compose --env-file .env.uat -f docker-compose.yml -f docker-compose.prod.yml up -d web worker beat
# entrypoint.sh runs migrate automatically on the new web container

# check it worked
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs web --tail 50
# look for "Applying <app>.<migration>... OK"
```

One-off manual migrate without a full rebuild/redeploy:
```bash
docker compose --env-file .env.uat -f docker-compose.yml -f docker-compose.prod.yml exec web python manage.py migrate
```

## DBeaver setup (SSH tunnel)

1. **Convert the `.ppk` key** — DBeaver's SSH tunnel needs OpenSSH format,
   not PuTTY's `.ppk`. In PuTTYgen: Load `.ppk` → **Conversions → Export
   OpenSSH key** → save as `key.pem`.

2. **New Connection → PostgreSQL**

3. **Main tab:**
   | Field | Value |
   |---|---|
   | Host | `localhost` |
   | Port | `5432` (prod) or `5433` (uat) |
   | Database / User / Password | from `.env.uat` on the server |

4. **SSH tab:**
   - Use SSH Tunnel: ✅
   - Host/IP: `<ec2-public-ip>`
   - Port: `22`
   - User: `ubuntu`
   - Auth: Public Key → browse to `key.pem`

5. Click **Test Tunnel Configuration** first (only checks SSH auth works),
   then **Test Connection** (actually checks postgres is reachable through
   it). Getting a green tunnel test but a failed DB test usually means
   nothing's listening on the target port yet — check `ss -tlnp` on the
   server side.

6. For the second environment, duplicate the connection, change only the
   Main tab port (5432 ↔ 5433) — SSH tab settings stay identical.

## Troubleshooting quick reference

| Symptom | Cause |
|---|---|
| `Test-NetConnection` / `nc` times out to port 5432 from local machine | Expected if using the tunnel approach — port intentionally not public |
| DBeaver: tunnel test OK, DB test `EOFException` | Nothing listening on the target port on the EC2 host yet — check `ss -tlnp`, likely need to apply/recreate the `db` container |
| `docker compose up -d db` says `Running` not `Recreated` | Compose sees no config change — usually means the compose file on the server is stale (forgot `git pull`) |
| `.env.uat` edit didn't take | Exited nano with Ctrl+C (discards) instead of Ctrl+O then Ctrl+X (saves) |
