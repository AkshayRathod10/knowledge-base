# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A knowledge base of operational runbooks and per-project deployment notes. There is no application code here — no build, lint, or test commands apply. Everything is Markdown.

## Structure and the generic/specific split

- `docker/` and `networking/` hold **generic, project-agnostic runbooks** — written with placeholders (`<ec2-host>`, `<env>`, `<remote-db-port>`, etc.) so the same doc applies to any project using that pattern (e.g. any Docker Compose project, any SSH-tunneled DB).
- `projects/<project-name>/` holds **notes for one specific project** that record the *actual* values and decisions made when applying a generic runbook there (real ports, real file paths, real commands run, gotchas hit in practice).

When adding a new runbook, keep this split: if it's reusable across projects, it belongs in a top-level category dir with placeholders; if it's "here's what we actually did for project X," it belongs under `projects/<project-name>/`.

## Conventions shared across the runbooks

- Every procedural step includes a **Verify** — the docs are written so you confirm each step worked before moving to the next, rather than chaining commands and discovering a failure several steps later at the wrong layer.
- Docs include a **Common pitfalls / Troubleshooting** table mapping symptom → cause, built from problems actually hit (not hypothetical ones).
- SSH tunneling is the established pattern for reaching a remote DB, in preference to opening its port in a Security Group — this is a deliberate security decision recorded in `networking/SSH_TUNNEL_DB_ACCESS_GUIDE.md`, not just one option among several.
- PuTTY `.ppk` keys must be converted to OpenSSH format (via PuTTYgen → Conversions → Export OpenSSH key) before use with DBeaver or other cross-platform SSH tunnel clients — `.ppk` is not portable outside PuTTY itself.
- Docker Compose projects here use a **base file + environment overlay** pattern (`docker-compose.yml` + `docker-compose.<env>.yml`), and `${VAR}` substitution in the compose YAML is resolved by Compose itself from `--env-file` at parse time — a separate mechanism from a service's `env_file:` directive, which only affects that container's runtime environment. Getting these confused is a recurring source of "why didn't my env change take effect."
- `docker compose up -d` does not rebuild images automatically — a source change needs an explicit `build` step first, or the running container keeps old behavior despite `git pull` and `up -d` both succeeding.
- After `git pull` + `up -d`, check the command's own output for `Recreated` vs `Running` per service — `Running` on a service you know changed means Compose saw no diff (stale pull, missed rebuild, or wrong directory), not that the deploy succeeded.
