# mc-server

A self-hosted Minecraft server stack running in Docker, with a modular set of
sidecar containers for console access, backups, file management, and basic
monitoring — plus an optional public-access path that works behind CGNAT (no
port forwarding, no static IP) via [playit.gg](https://playit.gg).

## Why this structure

[Crafty Controller](https://craftycontrol.com/) was evaluated as an
all-in-one web panel, but caused a split-ownership problem: when Crafty
"imports" a server that's actually running in a separate container, Crafty's
start/stop/console controls don't actually control the real process, and the
server never reliably showed up in its dashboard. See
[`docs/decisions.md`](docs/decisions.md) for the full writeup.

Instead, this repo uses the **`itzg` Docker ecosystem** as the single source
of truth for the running server (fully reproducible from a compose file +
`.env`), with small, independent sidecars bolted on for the specific
capabilities a panel like Crafty would otherwise provide:

| Capability | Tool |
|---|---|
| Server itself | [`itzg/minecraft-server`](https://github.com/itzg/docker-minecraft-server) (Java) or [`itzg/minecraft-bedrock-server`](https://github.com/itzg/docker-minecraft-bedrock-server) |
| Web console | [`itzg/rcon`](https://github.com/itzg/docker-rcon) (RCON Web Admin) — Java only, see note below |
| Scheduled backups | [`itzg/mc-backup`](https://github.com/itzg/docker-mc-backup) — Java only, see note below |
| Web file manager | [`filebrowser/filebrowser`](https://github.com/filebrowser/filebrowser) |
| Player count / status metrics | [`itzg/mc-monitor`](https://github.com/itzg/mc-monitor) — Java only, see note below |
| Public access behind CGNAT | [`playit.gg`](https://playit.gg) agent |

## Four compose files, one per combination

There's no single compose file — edition (Java/Bedrock) and public access
(with/without playit.gg) are independent choices, so each combination has
its own file under [`compose/`](compose/):

| File | Edition | Public access |
|---|---|---|
| `compose/docker-compose.java.yml` | Java | No (local/LAN only) |
| `compose/docker-compose.java.playit.yml` | Java | Yes (playit.gg) |
| `compose/docker-compose.bedrock.yml` | Bedrock | No (local/LAN only) |
| `compose/docker-compose.bedrock.playit.yml` | Bedrock | Yes (playit.gg) |

**Why Bedrock's files are leaner (no rcon-web, mc-backup, mc-monitor):**
those three sidecars all depend on **RCON**, which the Bedrock server does
not support (Bedrock has no RCON protocol implementation). Bedrock's
compose files include only `bedrock` + `filebrowser`. See
[`docs/bedrock.md`](docs/bedrock.md) for Bedrock-specific config, the
UDP/RakNet constraint, and why that also shapes the networking options.

Run whichever matches your situation:
```bash
cp .env.example .env   # fill in real values first — see below
docker compose -f compose/docker-compose.java.yml up -d
# or
docker compose -f compose/docker-compose.bedrock.playit.yml up -d
```

## Quick start

```bash
git clone <this-repo>
cd mc-server
cp .env.example .env
# edit .env — set real passwords; leave PLAYIT_SECRET_KEY blank unless
# using one of the *.playit.yml files
docker compose -f compose/docker-compose.java.yml up -d
```

- **Join the server (same machine/LAN):** Java launcher → Multiplayer → Add
  Server → `localhost:25565` (or your LAN IP from another device). For
  Bedrock, use the mobile/console app → Servers → Add Server →
  `<LAN IP>:19132`.
- **Web console (Java only):** `http://localhost:4326` (RCON Web Admin —
  login from `.env`).
- **File browser:** `http://localhost:8081`.
- **Player/status metrics (Java only):** `http://localhost:8080`
  (Prometheus-scrapeable).

## Going public (CGNAT-friendly)

This network is behind Carrier-Grade NAT — no port forwarding or dynamic DNS
is possible. Public access is handled by an outbound-only
[playit.gg](https://playit.gg) tunnel agent instead, included in the
`*.playit.yml` files. See [`docs/networking.md`](docs/networking.md) for the
full comparison of options considered (Cloudflare Tunnel, Cloudflare
Spectrum, frp, Tailscale/ZeroTier) and why playit.gg was chosen, plus setup
steps for the tunnel itself.

## Changing the Minecraft version

Edit the `VERSION` env var under the `mc` (or `bedrock`) service in whichever
compose file you're using, then re-run `docker compose -f <file> up -d`. The
image detects the version change and re-downloads the matching server binary
on next start — your world (on the bind-mounted `./data` or `./bedrock-data`
volume) is reused as-is. See [`docs/operations.md`](docs/operations.md) for
backup guidance before version jumps.

## Repo layout

```
mc-server/
├── compose/
│   ├── docker-compose.java.yml            # Java, local only
│   ├── docker-compose.java.playit.yml     # Java, + playit.gg
│   ├── docker-compose.bedrock.yml         # Bedrock, local only
│   └── docker-compose.bedrock.playit.yml  # Bedrock, + playit.gg
├── .env.example             # copy to .env, fill in real secrets
├── .gitignore
├── docs/
│   ├── decisions.md          # why Crafty was dropped, why this structure
│   ├── bedrock.md             # Bedrock-specific config + RCON/UDP caveats
│   ├── networking.md          # CGNAT, playit.gg, and the options ruled out
│   └── operations.md          # day-2 ops: versions, backups, users, troubleshooting
├── data/                     # (gitignored) Java world + server files
├── bedrock-data/             # (gitignored) Bedrock world + server files
└── backups/                  # (gitignored) scheduled backup output
```
