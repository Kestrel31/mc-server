# Operations

Day-2 tasks: changing versions, updating containers, backups, troubleshooting
auth, and sidecar-specific notes.

All commands below assume you're running one of the four files under
`compose/` with `-f`, e.g.:
```bash
docker compose -f compose/docker-compose.java.yml up -d
```
Substitute whichever of the four matches your edition/networking choice.
Tip: `export COMPOSE_FILE=compose/docker-compose.java.yml` in your shell
session lets you drop the `-f` flag for the rest of that session.

## Changing the Minecraft version

```bash
# edit VERSION under the `mc` (or `bedrock`) service in the compose file you use
docker compose -f compose/docker-compose.java.yml up -d
```

The image detects the version change on container start and downloads the
matching server binary into `/data` — no rebuild needed, it's fetched at
runtime, not baked into the image. The world on the bind-mounted volume is
reused as-is.

- **Upgrading** is generally safe — Minecraft auto-upgrades the world on
  first load with a newer server version.
- **Downgrading** is risky and can corrupt chunks or fail to load,
  especially on Bedrock. Back up first (see below) before any version
  change, and avoid downgrades unless you have a tested backup.
- `VERSION: "LATEST"` means every restart could silently pick up a newer
  release if one shipped in between — pin an exact version string for
  predictability.

## Applying any compose file change

```bash
docker compose -f compose/docker-compose.java.yml up -d
```

Compose diffs your YAML against the running containers and only recreates
what changed — untouched services are left alone. If you changed an image
tag and want to force a fresh pull rather than reuse a cached image:

```bash
docker compose -f compose/docker-compose.java.yml pull
docker compose -f compose/docker-compose.java.yml up -d
```

Verify:
```bash
docker compose -f compose/docker-compose.java.yml ps
docker compose -f compose/docker-compose.java.yml logs -f mc
```

`docker compose down -v` also removes named volumes — **do not** run this
casually; it would wipe sidecar state like `rcon-web-data`. Your world is
safe either way since it's on the external bind mount (`./data` or
`./bedrock-data`), not a Docker-managed volume.

## Switching between local and playit.gg (same edition)

Since the service names, volumes, and ports line up between
`docker-compose.java.yml` ↔ `docker-compose.java.playit.yml` (and the
Bedrock equivalents), you can stop one and start the other against the
same bind-mounted data without losing anything:

```bash
docker compose -f compose/docker-compose.java.yml down
docker compose -f compose/docker-compose.java.playit.yml up -d
```

## Backups

**Java**: handled automatically by the `mc-backup` sidecar
(`BACKUP_INTERVAL: "6h"`, pruned after `PRUNE_BACKUPS_DAYS: "7"`). It talks
to `mc` over RCON to safely pause world-saving during the backup, then
archives to `./backups`.

To force a backup immediately:
```bash
docker exec mc-backup backup-now
```

**Bedrock**: no automated sidecar in this repo (no RCON support to build
one on — see [`bedrock.md`](bedrock.md)). Back up `./bedrock-data`
manually, e.g.:
```bash
tar -czf backups/bedrock-$(date +%Y%m%d-%H%M%S).tar.gz bedrock-data/
```
or wire up your own scheduled job (cron, Task Scheduler, etc.) running
that same command periodically.

Always take a manual backup before a version change or any risky config
edit, even on Java where the schedule covers routine cases.

## Web console (RCON Web Admin) — Java only

`http://localhost:4326` — login with `RCON_WEB_ADMIN_USER` /
`RCON_WEB_ADMIN_PASSWORD` from `.env`. Gives live console + command input
without `docker exec`/`docker attach`.

Common commands to run from there:
```
op PlayerUsername              # grant operator/admin in-game
whitelist add PlayerUsername   # allow a player to join when whitelist is on
```

For Bedrock, there's no web console — use:
```bash
docker exec -it bedrock send-command op PlayerUsername
docker logs -f bedrock
```

## File browser (both editions)

`http://localhost:8081` — browse/edit `server.properties`, drop in plugin
jars (Java) or inspect world files, all against the same bind mount the
server uses. Supports multiple accounts with per-user permission scoping
if you want to give someone limited access without sharing the RCON/web
console credentials.

## Player count / status metrics — Java only

`mc-monitor` exposes a Prometheus-scrapeable endpoint at
`http://localhost:8080`. Pair with Prometheus + Grafana if you want
historical graphs (player count over time, uptime) — not included in this
repo by default to keep the base stack lean; add as a further sidecar if
wanted. No equivalent included for Bedrock.

## "Failed to verify username" (Java)

The server is in online mode (`ONLINE_MODE: "TRUE"`) and couldn't verify
your session with Mojang. Common causes, roughly in order of likelihood:

1. **Non-premium/offline launcher** — the account logged into the Java
   launcher isn't a genuine purchased Microsoft/Mojang account. This is by
   far the most common cause.
2. **Server can't reach Mojang's auth servers** — check `docker compose
   -f compose/docker-compose.java.yml logs mc` for timeout errors around
   authentication; the container needs outbound internet access.
3. **Clock skew** — compare `docker exec mc date` against actual current
   time; session verification is time-sensitive.
4. **Mojang outage** — check `https://status.mojang.com`.

For local/private testing only, you can bypass this by setting
`ONLINE_MODE: "FALSE"` — but this disables all identity verification
(anyone can log in as any username, including impersonating an op).
**Never** leave this on once the server is exposed publicly via playit.gg
unless it's also locked down with a whitelist.

## Crafty, if reintroduced later

If a web panel is ever added back purely for console/backup/file-browsing
convenience (not as the process owner — see
[`decisions.md`](decisions.md)):

- First boot generates a default admin account; check the log line it
  prints, or read the credentials file directly:
  ```bash
  docker exec -it crafty cat /crafty/app/config/default-creds.txt
  ```
  Change the password immediately after first login.
- To create additional panel users: **Panel Config → Users → Add User**,
  assign roles/permissions per-server rather than blanket admin access.
- To add an in-game player/op, that's unrelated to Crafty accounts — use
  its **Console** tab and run standard server commands (`op`, `whitelist
  add`), same as the RCON Web Admin approach above.
- If Crafty doesn't show an imported server in its dashboard, remember it
  tracks servers in its own database, not by scanning folder contents —
  it must be explicitly created or imported through its UI, and even then
  Start/Stop/Console won't control a process it didn't spawn itself. This
  is exactly the limitation documented in `decisions.md`.
