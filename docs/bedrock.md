# Bedrock Edition

Bedrock is a separate server implementation and protocol from Java Edition —
not a config flag on the same server. Use `compose/docker-compose.bedrock.yml`
(local only) or `compose/docker-compose.bedrock.playit.yml` (with public
access via playit.gg) instead of the `java` ones.

## No RCON — why Bedrock's compose files are leaner

The Java compose files include `rcon-web` (web console), `mc-backup`
(scheduled backups), and `mc-monitor` (player/status metrics) — all three
work by talking to the server over **RCON**. The Bedrock server has no
RCON implementation at all, so none of those three sidecars have anything
to connect to. Bedrock's compose files therefore only include `bedrock` +
`filebrowser` (which just reads the bind-mounted files directly, no RCON
needed).

Practical implications:
- **Console access**: use `docker attach bedrock` or `docker logs -f
  bedrock`, or send commands via `docker exec -it bedrock send-command
  <command>` (supported by the `itzg/minecraft-bedrock-server` image's
  helper scripts) — there's no web console equivalent for Bedrock.
- **Backups**: not automated in this repo for Bedrock. Either back up
  `./bedrock-data` manually/on a cron job (plain file copy, no RCON pause
  needed — there's no live world-save-lock requirement here the way
  `itzg/mc-backup` handles for Java), or adapt a simple scheduled `tar`
  job yourself.
- **Monitoring**: `mc-monitor` *does* work for Bedrock (it just does a
  status ping, no RCON needed) via its `status-bedrock` subcommand /
  `-bedrock-servers` flag on `export-for-prometheus` — it isn't wired up
  in `compose/docker-compose.bedrock.yml` by default, but could be added
  the same way it's used in the Java files (see `docs/operations.md`).

## Compose service (as used in `compose/docker-compose.bedrock.yml`)

```yaml
services:
  bedrock:
    image: itzg/minecraft-bedrock-server
    container_name: bedrock
    ports:
      - "19132:19132/udp"   # Bedrock is UDP (RakNet), not TCP — see below
    environment:
      EULA: "TRUE"
      VERSION: "LATEST"       # or a pinned version string, or "PREVIEW"
      GAMEMODE: "survival"
      DIFFICULTY: "normal"
      SERVER_NAME: "My Bedrock Server"
      LEVEL_NAME: "MyWorld"
      MAX_PLAYERS: "10"
      ALLOW_CHEATS: "false"
      DEFAULT_PLAYER_PERMISSION_LEVEL: "member"
      VIEW_DISTANCE: "32"
      ONLINE_MODE: "true"
    volumes:
      - ../bedrock-data:/data
    restart: unless-stopped
    tty: true
    stdin_open: true
```

Nearly every `server.properties` setting has a matching env var — see the
[itzg/docker-minecraft-bedrock-server docs](https://github.com/itzg/docker-minecraft-bedrock-server)
for the full list (~40+ options: whitelist, texture pack enforcement, seed,
Xbox Live auth, etc.).

You can also hand-edit files directly on the bind-mounted volume (or via
the `filebrowser` sidecar at `http://localhost:8081`):
- `server.properties` — core settings (takes effect on container restart)
- `permissions.json` — per-player permission levels
- `allowlist.json` — whitelist entries
- `worlds/<LEVEL_NAME>/` — the actual world data

## Allowlist ("not invited to join")

`allow-list` defaults to `true` in the Bedrock server — meaning **nobody
can join unless explicitly listed**, confirmed by inspecting a fresh
container's generated `server.properties`. This is almost certainly why a
player sees "you are not invited to join" even though the server itself is
reachable.

All of this repo's Bedrock compose files set:
```yaml
environment:
  ALLOW_LIST: "false"
```
which disables the allowlist entirely — anyone who reaches the server can
join. Confirm it applied:
```bash
docker exec bedrock cat /data/server.properties | grep ^allow-list
# should print: allow-list=false
```

**If you'd rather keep the allowlist on** and explicitly approve players
instead, set `ALLOW_LIST: "true"` and add entries to `allowlist.json`
(mounted at `./bedrock-data/allowlist.json`, editable via the `filebrowser`
sidecar or directly):
```json
[
  { "name": "PlayerGamertag", "ignoresPlayerLimit": false }
]
```
Takes effect on next container restart — no need to touch `server.properties`
directly for this one, just the JSON file.

## Important: Bedrock is UDP, not TCP — and that's not configurable

Bedrock clients (Windows, console, mobile — all of them) use **RakNet**,
which is built on UDP at the protocol level inside the client itself. There
is no server-side setting, env var, or config flag anywhere in the
ecosystem that switches this to TCP. This has real consequences, and is
exactly why there are separate `*.playit.yml` files rather than one
networking approach for both editions:

## NetherNet vs. RakNet — a second transport layer, and why it matters here

Newer Bedrock server builds default to a transport called **NetherNet**
instead of classic RakNet. NetherNet is Microsoft's newer transport, built
around Xbox Live's own signaling/relay (used for the "Friends"/cross-device
matchmaking flow) — it is **not** a plain UDP socket a third-party tunnel
can forward. playit.gg (and effectively every other third-party
tunnel/relay) can only forward classic RakNet traffic on a fixed UDP port.

The `itzg/minecraft-bedrock-server` image's current default is
`transport=nethernet`, confirmed by inspecting a fresh container's
generated `server.properties`. All four of this repo's compose files
explicitly set:
```yaml
environment:
  TRANSPORT: "raknet"
```
to override that default. Without this, the server runs fine for players
on Xbox Live/cross-device matchmaking, but playit.gg (or any other
UDP-port-forwarding tunnel) has nothing to actually forward — there's no
traditional RakNet socket open at all, which is exactly the "switch your
server to ranked [sic — RakNet]" message playit.gg's own setup flow
surfaces.

Confirm it took effect:
```bash
docker exec bedrock cat /data/server.properties | grep ^transport
# should print: transport=raknet
```

With that in mind:

- **Cloudflare Tunnel's TCP ingress** (what you'd use for the Java server
  or for a web panel's UI) does **not** apply to Bedrock traffic at all —
  Bedrock never sends TCP.
- **Cloudflare Tunnel's UDP support** exists, but only for WARP-enrolled
  private network access — every player would need to install Cloudflare
  WARP and be added to your Zero Trust org. Not viable for public players.
- **Cloudflare Spectrum** is the only Cloudflare product that proxies
  public UDP transparently (and explicitly lists Minecraft Bedrock as a
  supported use case) — but it's a paid add-on, not available on the free
  tier.
- **playit.gg** (see [`networking.md`](networking.md)) supports UDP on its
  free tier and is used in `docker-compose.bedrock.playit.yml` —
  especially relevant since this network is behind CGNAT and port
  forwarding isn't possible at all, making a tunnel mandatory rather than
  optional for Bedrock specifically.

Theoretically you could wrap UDP inside a TCP tunnel yourself
(`udp2raw` or similar), but every player's device would need that same
wrapper running locally before the game even starts — not practical for
anyone but yourself.

## Cross-play notes

- Java and Bedrock clients cannot connect to each other's servers directly
  — different protocols entirely. A Java server only accepts Java clients;
  a Bedrock server only accepts Bedrock clients.
- To let Bedrock clients join a **Java** server instead of running a
  separate Bedrock server, use the **Geyser + Floodgate** plugin pair on a
  Paper server. This can simplify the whole stack to a single Java server
  (and regain RCON-based console/backups/monitoring) if cross-play is the
  actual goal rather than "I only have the Bedrock app handy."
- Bedrock world/version compatibility is stricter than Java's across
  version jumps, and downgrading is riskier (more likely to corrupt or fail
  to load). Always back up `./bedrock-data` before changing `VERSION`.
- Client/server version alignment matters more on Bedrock — especially
  across console/mobile, which auto-update — mismatches often cause
  outright connection failures rather than a tolerated resource-pack
  negotiation like Java.

## Local testing without port-forwarding

Bedrock mobile clients can test over plain LAN without any tunnel, as long
as both devices share the same Wi-Fi network (not a guest/isolated network,
and not different networks/cellular):

1. Find your PC's LAN IP: `ipconfig` (Windows) → IPv4 Address.
2. In the Bedrock app: Play → Servers → Add Server → address = that IP,
   port = `19132`.
3. If it can't connect, check Windows Defender Firewall → Inbound Rules →
   allow UDP port `19132`.
4. Confirm the container is actually publishing the port:
   `docker ps` should show `0.0.0.0:19132->19132/udp` for the bedrock
   container.

This is also a good dry run for how playit.gg will later route traffic to
the same port, just from the internet instead of the LAN, once you switch
to `compose/docker-compose.bedrock.playit.yml`.
