# Accessing a Remote Database via SSH Tunnel

Generic runbook for reaching a database (Postgres, MySQL, etc.) that's
running on a remote host — EC2, bare VM, anywhere you have SSH access — 
**without** opening its port in a Security Group / firewall to the public
internet. Works for any DB client (DBeaver, psql, pgAdmin, TablePlus...).
Placeholders: `<ec2-user>`, `<ec2-host>`, `<remote-db-port>`,
`<local-port>` (usually same as `<remote-db-port>` for simplicity).

Every step includes a **Verify** — each layer (SSH auth, remote listener,
tunnel, DB auth) can fail independently, and the error messages for each
failure mode look different. Verifying at each layer tells you exactly
which one broke instead of guessing.

---

## Why tunnel instead of opening the port

| | Open port in SG | SSH tunnel |
|---|---|---|
| DB port reachable from internet | Yes (gated by SG rule + your IP) | No — never touches the network interface |
| Setup | SG rule change (often needs infra/devops access) | Nothing to change on the network side |
| Risk if your IP changes / SG rule too broad | DB exposed | No change in exposure |
| Works through NAT/no-static-IP situations | Needs updating the SG rule each time | Yes — tunnel just needs SSH access, which you already have |

If you only have SSH access (key + host, no console/IAM access) — which is
common for a shared or third-party-managed instance — the tunnel is often
the *only* option available anyway, not just the safer one.

---

## 0. Prerequisites

- SSH access to the host (key file, correct username)
- The DB is running and listening on **at least `127.0.0.1:<remote-db-port>`**
  on the remote host (it does not need to listen on `0.0.0.0` — loopback
  is enough, since the tunnel terminates locally on that same host)

**Verify DB is actually listening before attempting a tunnel** — this is
the single most common wasted-time step (chasing a tunnel/auth problem
when the real issue is nothing's listening at all):
```bash
ssh -i key.pem <ec2-user>@<ec2-host> "sudo ss -tlnp | grep <remote-db-port>"
```
Expect a `LISTEN` line with `127.0.0.1:<remote-db-port>` (or `0.0.0.0:...`).
**Empty output = stop here.** Fix the DB/container first; a tunnel to a
port nothing is listening on will fail with a confusing error later
(commonly `EOFException` in JDBC-based tools, or the tunnel silently doing
nothing) rather than a clear "nothing there."

---

## 1. Convert your key if needed

PuTTY's `.ppk` format is **not** read by most cross-platform SSH tunnel
implementations (DBeaver's built-in JSch, most Java/Node SSH libraries).
OpenSSH-format keys are the portable choice.

**If you have a `.ppk`:**
1. Open **PuTTYgen**
2. **Load** the `.ppk`
3. Menu → **Conversions → Export OpenSSH key** → save as `key.pem`

**Verify:**
```bash
head -1 key.pem
```
Should read `-----BEGIN OPENSSH PRIVATE KEY-----` or
`-----BEGIN RSA PRIVATE KEY-----` — not PuTTY's own header format.

On Linux/macOS, also fix permissions or SSH will refuse the key outright:
```bash
chmod 600 key.pem
```

---

## 2. Test raw SSH access works, standalone, before adding tunneling

```bash
ssh -i key.pem <ec2-user>@<ec2-host> "echo connected"
```

**Verify:** prints `connected`. If this fails, nothing downstream
(tunnel, DB client) will work either — fix SSH auth first (wrong key,
wrong user, wrong host, SG blocking port 22, key permissions).

---

## 3. Open the tunnel manually from the command line first

Even if you'll end up using a GUI tool, proving the tunnel works via CLI
first isolates "is the tunnel broken" from "is the GUI tool configured
wrong" — two very different failure classes.

```bash
ssh -i key.pem -L <local-port>:localhost:<remote-db-port> <ec2-user>@<ec2-host>
```
Leave this session open — the tunnel only exists while it's connected.

**Verify, in a second local terminal:**
```bash
# Windows PowerShell
Test-NetConnection -ComputerName localhost -Port <local-port>

# macOS/Linux
nc -zv localhost <local-port>
```
Should succeed immediately (unlike testing the remote IP directly, which
timed out before the tunnel existed — that's expected and correct, the
port was never meant to be reachable that way).

**Stronger verify — actually speak the DB protocol through it**, if the
client is installed locally:
```bash
psql -h localhost -p <local-port> -U <db-user> -d <db-name>
```
A password prompt (even if you don't have credentials handy yet) confirms
the tunnel is correctly forwarding to a real Postgres instance, not just
an open TCP port. Ctrl+C / Ctrl+D out once confirmed.

---

## 4. Configure the tunnel in your DB client (example: DBeaver)

1. New Connection → your DB type (e.g. PostgreSQL)
2. **Main tab:**
   | Field | Value |
   |---|---|
   | Host | `localhost` |
   | Port | `<remote-db-port>` |
   | Database / User / Password | actual DB credentials |
3. **SSH tab:**
   - Use SSH Tunnel: ✅
   - Host/IP: `<ec2-host>`
   - Port: `22`
   - User: `<ec2-user>`
   - Auth: Public Key → `key.pem`

**Verify in two separate stages — they test different things:**
- **Test Tunnel Configuration** — only proves SSH auth works (same as
  step 2). A green result here does **not** mean the DB is reachable.
- **Test Connection** — proves the DB is actually reachable and accepting
  auth through the tunnel. This is the one that matters.

If tunnel test passes but connection test fails with something like
`EOFException` or "connection reset": go back to step 0's verify — almost
always means nothing's listening on `<remote-db-port>` on the remote side.

---

## 5. Multiple databases / ports in one setup

If tunneling to more than one DB on the same host (e.g. separate `uat` and
`prod` instances each bound to their own loopback port), forward both in
one SSH session rather than opening two:
```bash
ssh -i key.pem -L 5432:localhost:5432 -L 5433:localhost:5433 <ec2-user>@<ec2-host>
```
In the DB client, create one connection per DB, same SSH tab settings,
differing only in the Main tab's port.

---

## Troubleshooting

| Symptom | Layer | Cause |
|---|---|---|
| `ssh` itself fails / times out | Network / SSH | Wrong host/IP, SG blocking port 22, wrong key |
| `ssh` fails with permission denied | Auth | Wrong username, wrong key, key not converted from `.ppk`, key file permissions too open (needs `chmod 600` on Linux/macOS) |
| Tunnel test (SSH) passes, DB test fails, `EOFException` / connection reset | Remote listener | Nothing listening on `<remote-db-port>` on the remote host — check `ss -tlnp` there |
| DB test fails with an actual auth error (not EOF/timeout) | DB auth | Tunnel + listener are fine — wrong DB username/password/database name |
| `Test-NetConnection`/`nc` to the **public IP** directly times out | Expected | This is correct behavior if the DB is intentionally not exposed — that's the whole point of tunneling instead |
| Tunnel works, then randomly drops | SSH session | The manual `-L` tunnel only lives as long as that SSH session stays connected — closing the terminal kills it; GUI tools manage their own persistent tunnel per connection |
