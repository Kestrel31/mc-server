# Operations

Day-2 tasks: changing versions, updating containers, backups, troubleshooting
auth, and sidecar-specific notes.

## Known issues already fixed in this repo's history

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

2. **RCON Web Admin (`itzg/rcon`) showing no servers /
   `127.0.0.1:25575 connection failed`** — this sidecar had several
   stacked config mistakes (wrong env var for username, missing
   `RWA_ENV`/`RWA_RCON_*` vars, wrong host). Rather than keep layering
   fixes onto a sidecar that wasn't earning its place, it was **removed
   entirely** — see [`decisions.md`](decisions.md). Console access is now
   via `docker exec`/`docker attach` directly (see below). `ENABLE_RCON`
   stays on for `mc` itself since `mc-backup` still needs it internally.

3. **FileBrowser was `filebrowser/filebrowser`, which is now archived**
   (no further releases or security fixes). Replaced with the actively
   maintained fork **`gtstef/filebrowser`** ("FileBrowser Quantum",
   `github.com/gtsteffaniak/filebrowser`). This fork configures via a
   mounted `config.yaml` instead of `FB_USERNAME`/`FB_PASSWORD` env vars
   — see the File browser section below for setup.

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
casually; it would wipe sidecar state like `filebrowser-db`. Your world is
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

**Java**: handled automatically by the `mc-backup` sidecar —
`BACKUP_INTERVAL: "24h"` (daily), pruned after `PRUNE_BACKUPS_DAYS: "7"`.
It talks to `mc` over RCON to safely pause world-saving during the
backup, then archives to `./backups`.

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

## Console access (no web console)

RCON Web Admin was removed (see `decisions.md`) — use `docker exec`/
`docker attach` directly instead:

**Java:**
```bash
docker exec mc rcon-cli op PlayerUsername
docker exec mc rcon-cli whitelist add PlayerUsername
docker attach mc          # live console; Ctrl+P, Ctrl+Q to detach without stopping the server
docker compose -f compose/docker-compose.java.yml logs -f mc
```

**Bedrock:**
```bash
docker exec -it bedrock send-command op PlayerUsername
docker logs -f bedrock
```

## File browser

Config lives in `filebrowser.config.yaml` (mounted read-only into the
container), not environment variables — set on first setup:
```bash
cp filebrowser.config.yaml.example filebrowser.config.yaml
# edit filebrowser.config.yaml — set a real auth.adminPassword before first run
```

Then `http://localhost:8081` — login with `auth.adminUsername` /
`auth.adminPassword` from that file. To change the password later without
editing the file and restarting:
```bash
docker exec filebrowser filebrowser user set admin <new-password>
```

Browse/edit `server.properties`, drop in plugin jars (Java), or inspect
world files, all against the same bind mount the server uses.

## Player count / status metrics

`mc-monitor` exposes a Prometheus-scrapeable endpoint at
`http://localhost:8080`. Pair with Prometheus + Grafana if you want
historical graphs (player count over time, uptime) — not included in this
repo by default to keep the base stack lean; add as a further sidecar if
wanted. Wired in for Java only by default; the same tool works for
Bedrock too (`status-bedrock` / `-bedrock-servers`) if you want to add it
to `compose/docker-compose.bedrock.yml` yourself.

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
