# mc-server

A self-hosted Minecraft server stack running in Docker, with a modular set of
sidecar containers for backups, file management, and basic monitoring —
plus an optional public-access path that works behind CGNAT (no port
forwarding, no static IP) via [playit.gg](https://playit.gg).

## Why this structure

[Crafty Controller](https://craftycontrol.com/) was evaluated as an
all-in-one web panel, but caused a split-ownership problem: when Crafty
"imports" a server that's actually running in a separate container, Crafty's
start/stop/console controls don't actually control the real process, and the
server never reliably showed up in its dashboard. See
[`docs/decisions.md`](docs/decisions.md) for the full writeup.

A web console sidecar (`itzg/rcon` / RCON Web Admin) was also tried and
later removed — it added a config surface that never justified itself once
set up correctly. See [`docs/decisions.md`](docs/decisions.md).

Instead, this repo uses the **`itzg` Docker ecosystem** as the single source
of truth for the running server (fully reproducible from a compose file +
`.env`), with small, independent sidecars bolted on for the specific
capabilities that still matter:

| Capability | Tool |
|---|---|
| Server itself | [`itzg/minecraft-server`](https://github.com/itzg/docker-minecraft-server) (Java) or [`itzg/minecraft-bedrock-server`](https://github.com/itzg/docker-minecraft-bedrock-server) |
| Console access | `docker attach mc` / `docker exec mc rcon-cli <cmd>` (Java), `docker exec bedrock send-command <cmd>` (Bedrock) — no web console; see `docs/decisions.md` |
| Scheduled backups (daily) | [`itzg/mc-backup`](https://github.com/itzg/docker-mc-backup) — Java only, needs RCON internally (see note below) |
| Web file manager | [`gtsteffaniak/filebrowser`](https://github.com/gtsteffaniak/filebrowser) ("FileBrowser Quantum" — actively maintained fork; replaces the now-archived `filebrowser/filebrowser`) |
| Player count / status metrics | [`itzg/mc-monitor`](https://github.com/itzg/mc-monitor) — works for both editions (status ping only, no RCON needed) |
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

**Why Bedrock's files are leaner (no mc-backup, mc-monitor by default):**
`mc-backup` depends on **RCON**, which the Bedrock server does not support
(Bedrock has no RCON protocol implementation) — so Bedrock's compose files
include only `bedrock` + `filebrowser`. `mc-monitor` *could* be added for
Bedrock too (it only does a status ping, no RCON needed) but isn't wired
in by default. See [`docs/bedrock.md`](docs/bedrock.md) for Bedrock-specific
config, the UDP/RakNet constraint, and why that also shapes the networking
options.

## Quick start

```bash
git clone <this-repo>
cd mc-server

cp .env.example .env
# edit .env — set a real RCON_PASSWORD; leave PLAYIT_SECRET_KEY blank
# unless using one of the *.playit.yml files

cp filebrowser.config.yaml.example filebrowser.config.yaml
# edit filebrowser.config.yaml — set a real auth.adminPassword

docker compose -f compose/docker-compose.java.yml up -d
```

- **Join the server (same machine/LAN):** Java launcher → Multiplayer → Add
  Server → `localhost:25565` (or your LAN IP from another device). For
  Bedrock, use the mobile/console app → Servers → Add Server →
  `<LAN IP>:19132`.
- **File browser:** `http://localhost:8081` — login with the username/
  password set in `filebrowser.config.yaml`.
- **Player/status metrics:** `http://localhost:8080` (Prometheus-scrapeable).
- **Console access:** `docker exec mc rcon-cli <command>` (Java) or
  `docker exec bedrock send-command <command>` (Bedrock) — see
  [`docs/operations.md`](docs/operations.md).

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
├── .env.example                      # copy to .env, fill in real secrets
├── filebrowser.config.yaml.example   # copy to filebrowser.config.yaml, set a real password
├── .gitignore
├── docs/
│   ├── decisions.md          # why Crafty and RCON Web Admin were dropped
│   ├── bedrock.md             # Bedrock-specific config + RCON/UDP caveats
│   ├── networking.md          # CGNAT, playit.gg, and the options ruled out
│   └── operations.md          # day-2 ops: versions, backups, console, troubleshooting
├── data/                     # (gitignored) Java world + server files
├── bedrock-data/             # (gitignored) Bedrock world + server files
└── backups/                  # (gitignored) scheduled backup output
```
