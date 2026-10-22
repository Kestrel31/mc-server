# mc-server

A self-hosted Minecraft server stack running in Docker, with a modular set of
sidecar containers for console access, backups, file management, and basic
monitoring — plus a public-access path that works behind CGNAT (no port
forwarding, no static IP) via [playit.gg](https://playit.gg).

## Why this structure

[Crafty Controller](https://craftycontrol.com/) was evaluated as an
all-in-one web panel, but caused a split-ownership problem: when Crafty
"imports" a server that's actually running in a separate container, Crafty's
start/stop/console controls don't actually control the real process, and the
server never reliably showed up in its dashboard. See
[`docs/decisions.md`](docs/decisions.md) for the full writeup.

Instead, this repo uses the **`itzg` Docker ecosystem** as the single source
of truth for the running server (fully reproducible from `docker-compose.yml`
+ `.env`), with small, independent sidecars bolted on for the specific
capabilities a panel like Crafty would otherwise provide:

| Capability | Tool |
|---|---|
| Server itself | [`itzg/minecraft-server`](https://github.com/itzg/docker-minecraft-server) (Java) or [`itzg/minecraft-bedrock-server`](https://github.com/itzg/docker-minecraft-bedrock-server) |
| Web console | [`itzg/rcon`](https://github.com/itzg/docker-rcon) (RCON Web Admin) |
| Scheduled backups | [`itzg/mc-backup`](https://github.com/itzg/docker-mc-backup) |
| Web file manager | [`filebrowser/filebrowser`](https://github.com/filebrowser/filebrowser) |
| Player count / status metrics | [`itzg/mc-monitor`](https://github.com/itzg/mc-monitor) |
| Public access behind CGNAT | [`playit.gg`](https://playit.gg) agent |

## Quick start

```bash
git clone <this-repo>
cd mc-server
cp .env.example .env
# edit .env — set real passwords, leave PLAYIT_SECRET_KEY blank for now
docker compose up -d
```

- **Join the server (same machine/LAN):** Java launcher → Multiplayer → Add
  Server → `localhost:25565` (or your LAN IP from another device).
- **Web console:** `http://localhost:4326` (RCON Web Admin — login from `.env`).
- **File browser:** `http://localhost:8081`.
- **Player/status metrics:** `http://localhost:8080` (Prometheus-scrapeable).

## Switching to Bedrock

Bedrock uses a different image, a different protocol (UDP 19132 via RakNet —
not configurable to TCP), and different config env vars. See
[`docs/bedrock.md`](docs/bedrock.md) for the swapped-in service definition
and the public-access implications (Cloudflare Tunnel's TCP ingress does
**not** work for Bedrock).

## Going public (CGNAT-friendly)

This network is behind Carrier-Grade NAT — no port forwarding or dynamic DNS
is possible. Public access is handled by an outbound-only
[playit.gg](https://playit.gg) tunnel agent instead. See
[`docs/networking.md`](docs/networking.md) for the full comparison of options
considered (Cloudflare Tunnel, Cloudflare Spectrum, frp, Tailscale/ZeroTier)
and why playit.gg was chosen.

## Changing the Minecraft version

Just edit the `VERSION` env var under the `mc` service in
`docker-compose.yml`, then:

```bash
docker compose up -d
```

The image detects the version change and re-downloads the matching server
jar on next start — your world (on the bind-mounted `./data` volume) is
reused as-is. See [`docs/operations.md`](docs/operations.md) for backup
guidance before version jumps, and `docker compose pull` notes for image
tag updates.

## Repo layout

```
mc-server/
├── docker-compose.yml     # Java edition stack (default)
├── .env.example            # copy to .env, fill in real secrets
├── .gitignore
├── docs/
│   ├── decisions.md         # why Crafty was dropped, why this structure
│   ├── bedrock.md            # Bedrock-specific compose + caveats
│   ├── networking.md         # CGNAT, playit.gg, and the options ruled out
│   └── operations.md         # day-2 ops: versions, backups, users, troubleshooting
├── data/                    # (gitignored) world + server files, bind-mounted
└── backups/                 # (gitignored) scheduled backup output
```
