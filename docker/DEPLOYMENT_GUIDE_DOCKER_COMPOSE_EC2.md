# Deploying a Docker Compose Stack to an EC2 Instance

Generic runbook. Works for any project using the "base compose file +
environment-specific overlay" pattern (`docker-compose.yml` +
`docker-compose.<env>.yml`), deployed to a Linux EC2 instance with Docker
already installed. Placeholders to swap: `<project-dir>`, `<env>` (e.g.
`prod`, `uat`), `<env-file>` (e.g. `.env.prod`, `.env.uat`), `<ec2-user>`,
`<ec2-host>`.

Every step includes a **Verify** — don't move to the next step until the
verify passes. Skipping verification is how a bad deploy gets discovered
by users instead of by you.

---

## 0. Prerequisites

- SSH access to the instance (key file, correct user)
- Docker + Docker Compose v2 installed on the instance
- Deployment directory already exists on the instance with the repo cloned
  (first-time clone is a separate one-off step, not covered here)
- An environment file (`<env-file>`) already present in the deployment
  directory with the required variables for that environment

**Verify:**
```bash
ssh -i key.pem <ec2-user>@<ec2-host> "docker --version && docker compose version"
```
Both must return version strings, not "command not found."

---

## 1. Connect and locate the deployment directory

```bash
ssh -i key.pem <ec2-user>@<ec2-host>
cd <project-dir>
pwd
```

**Verify:**
```bash
ls docker-compose*.yml
```
Confirm the base file and the environment overlay you expect
(e.g. `docker-compose.yml`, `docker-compose.prod.yml`) both exist here.
If they don't, you're either in the wrong directory or the repo wasn't
cloned/synced yet.

---

## 2. Check current running state (baseline before touching anything)

```bash
docker compose -f docker-compose.yml -f docker-compose.<env>.yml ps
```

**Verify:** note which containers are currently `Up` and for how long.
This is your rollback reference point — if the deploy goes wrong, you
know what "before" looked like.

---

## 3. Pull the latest code

```bash
git status          # confirm no uncommitted local changes on this box
git pull
```

**Verify:**
```bash
git log -1 --oneline
git status
```
- `git log -1` should show the commit you expect to deploy
- `git status` should say "working tree clean" — if it doesn't, something
  was edited directly on the server (a config drift risk) and needs
  investigating before continuing, not silently overwritten

---

## 4. Confirm the environment file has what the new code expects

If the change added new required env vars, they need to already be in
`<env-file>` before containers start, or the app will fail at startup
(or worse, start with a wrong default).

```bash
grep -E 'VAR_NAME_1|VAR_NAME_2' <env-file>
```

**Verify:** every variable the new code/compose file references is present
and non-empty. Cross-check against the compose file's `${VAR}` /
`${VAR:-default}` references:
```bash
grep -oE '\$\{[A-Z_]+' docker-compose.<env>.yml | sort -u
```

---

## 5. Rebuild images (only needed if source code changed — not needed for a pure config/env change)

Compose does **not** rebuild automatically on `up -d`. If the app image
is built from local source (`build: context: ...`), changes only take
effect after an explicit rebuild.

```bash
docker compose --env-file <env-file> -f docker-compose.yml -f docker-compose.<env>.yml build <service1> <service2>
```

**Verify:**
```bash
docker images | head
```
Check the image for `<service1>` has a fresh `CREATED` timestamp
(seconds/minutes ago, not days).

---

## 6. Recreate containers

```bash
docker compose --env-file <env-file> -f docker-compose.yml -f docker-compose.<env>.yml up -d
```

**Verify — read the command's own output, don't just check it exited 0:**
- Services with real changes should say **`Recreated`**
- Services with no changes should say **`Running`** (unchanged, correctly
  left alone)
- If a service you *know* changed says `Running` instead of `Recreated`,
  compose didn't see a diff — usual cause: step 3's `git pull` didn't
  actually update the compose file, or step 5's rebuild didn't happen, or
  you're pointed at the wrong directory/file

Then:
```bash
docker compose -f docker-compose.yml -f docker-compose.<env>.yml ps
```
All services should show `Up` (or `Up (healthy)` if they have a
healthcheck). Compare against your step 2 baseline — anything that
dropped off or is restarting is a problem.

---

## 7. Check logs for startup errors

```bash
docker compose -f docker-compose.yml -f docker-compose.<env>.yml logs <service> --tail 100
```

**Verify:** no tracebacks, no "exited," no crash-loop. If the service runs
migrations or other startup steps (check its entrypoint script), confirm
those specific steps completed:
```bash
docker compose -f docker-compose.yml -f docker-compose.<env>.yml logs <service> --tail 100 | grep -i -E 'error|traceback|applying'
```

---

## 8. Smoke test

Hit a real endpoint, don't just trust "container is Up" — a container can
be running with a completely broken app inside it.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:<port>/api/health/
```

**Verify:** expected status code (e.g. `200`). If there's no health
endpoint, curl any known-cheap endpoint and check for a sane response,
not a 500 or connection refused.

---

## 9. If something's wrong — rollback

Data in named volumes is untouched by any of the above (container
recreation doesn't touch volumes) — rollback is safe from a data-loss
perspective.

```bash
git log --oneline -5          # find the last known-good commit
git checkout <previous-commit-or-tag>
docker compose --env-file <env-file> -f docker-compose.yml -f docker-compose.<env>.yml build <service>
docker compose --env-file <env-file> -f docker-compose.yml -f docker-compose.<env>.yml up -d <service>
```

**Verify:** repeat steps 6–8 against the rolled-back version.

---

## Common pitfalls

| Symptom | Cause |
|---|---|
| `up -d` says `Running` for a service you changed | Compose file on the server is stale — `git pull` step was skipped or targeted wrong branch |
| App container `Up` but serving old behavior | Source-built image wasn't rebuilt after `git pull` (prod/uat typically don't bind-mount source, unlike a dev override) |
| New env var not taking effect | `${VAR}` in the compose YAML is resolved by compose itself from `--env-file`/shell env — separate from a service's `env_file:` directive, which only affects the container's runtime env, not compose's own variable substitution |
| Edited env file but change didn't stick | Editor exited without saving (e.g. nano: Ctrl+C discards, Ctrl+O then Ctrl+X saves) |
| Container restarting in a loop after deploy | Check logs first (step 7) before doing anything else — usually a startup script failing (bad migration, missing env var), not a docker problem |
