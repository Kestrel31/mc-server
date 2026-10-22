# Operations

Day-2 tasks: changing versions, updating containers, backups, troubleshooting
auth, and sidecar-specific notes.

## Known issues already fixed in this repo's compose files

A few sidecar config mistakes were made and corrected during initial setup
— documenting them here since the symptoms are easy to hit again if you
fork/modify this stack:

1. **`mc-monitor` restart-looping** — the compose files originally used
   `command: export --host=mc`, but the actual subcommand is
   `export-for-prometheus` with a `-servers host:port` flag (space, not
   `=`). Fixed to:
   ```yaml
   command: ["export-for-prometheus", "-servers", "mc:25565"]
   ```
   Diagnose this kind of thing yourself with:
   ```bash
   docker run --rm itzg/mc-monitor:latest --help
   docker run --rm itzg/mc-monitor:latest help export-for-prometheus
   ```

2. **RCON Web Admin showing no servers / `127.0.0.1:25575 connection
   failed`** — two stacked mistakes:
   - `RWA_ADMIN` was being used to set the *username*, but it's actually a
     `TRUE`/`FALSE` flag for "make this user an admin." The username var is
     `RWA_USERNAME`.
   - The RCON *connection* itself was never configured at all —
     `RWA_ADMIN`/`RWA_PASSWORD` only configure the web panel's own login,
     not the connection to `mc`. `RWA_ENV: "TRUE"` must be set to enable
     env-var config, plus `RWA_RCON_HOST`, `RWA_RCON_PORT`, and
     `RWA_RCON_PASSWORD` to actually point it at the Minecraft server.
   - Separately: even with those set, the host must be the **Docker
     service name** (`mc`), never `127.0.0.1` or `localhost` — those
     resolve to the `rcon-web` container itself, not `mc`, since they're
     separate containers. `mc-backup`'s `RCON_HOST: mc` already did this
     correctly; `rcon-web` didn't.

   Current (correct) config:
   ```yaml
   environment:
     RWA_ENV: "TRUE"
     RWA_USERNAME: "${RCON_WEB_ADMIN_USER}"
     RWA_PASSWORD: "${RCON_WEB_ADMIN_PASSWORD}"
     RWA_ADMIN: "TRUE"
     RWA_GAME: "minecraft"
     RWA_SERVER_NAME: "mc"
     RWA_RCON_HOST: "mc"
     RWA_RCON_PORT: "25575"
     RWA_RCON_PASSWORD: "${RCON_PASSWORD}"
   ```
   Full env var reference: the [rcon-web-admin README](https://github.com/rcon-web-admin/rcon-web-admin#environment-variables)
   (this is the actively maintained fork; the original by @brainfoolong is
   no longer maintained).

3. **FileBrowser credentials** — never explicitly set, so it was silently
   falling back to its built-in default (`admin` / `admin`). Now set
   explicitly via `.env`:
   ```yaml
   environment:
     FB_USERNAME: "${FILEBROWSER_USER}"
     FB_PASSWORD: "${FILEBROWSER_PASSWORD}"
   ```
   **⚠️ Also: `filebrowser/filebrowser` is archived/unmaintained** (no
   further releases or security fixes as of its archive date). This repo
   still references it since it's a drop-in simple option, but consider
   swapping to the actively maintained fork
   [`gtsteffaniak/filebrowser`](https://github.com/gtsteffaniak/filebrowser)
   (same core functionality, different image name) if you want ongoing
   security patches.

4. **Running `docker compose` from inside `compose/` instead of the repo
   root** — since the compose files use relative volume paths like
   `../data`, and `.env` lives at the repo root, always run commands from
   the repo root with `-f compose/<file>.yml`, not `cd compose && ...`.
   Confirm `.env` is actually being picked up with:
   ```bash
   docker compose -f compose/docker-compose.java.yml config | grep RCON_PASSWORD
   ```
   If this prints the literal string `${RCON_PASSWORD}` instead of your
   real value, `.env` isn't being found.

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
