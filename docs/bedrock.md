# Bedrock Edition

Bedrock is a separate server implementation and protocol from Java Edition —
not a config flag on the same server. If you want Bedrock instead of (or
alongside) Java, swap in this service.

## Compose service

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
      - ./bedrock-data:/data
    restart: unless-stopped
    tty: true
    stdin_open: true
```

Nearly every `server.properties` setting has a matching env var — see the
[itzg/docker-minecraft-bedrock-server docs](https://github.com/itzg/docker-minecraft-bedrock-server)
for the full list (~40+ options: whitelist, texture pack enforcement, seed,
Xbox Live auth, etc.).

You can also hand-edit files directly on the bind-mounted volume:
- `server.properties` — core settings (takes effect on container restart)
- `permissions.json` — per-player permission levels
- `allowlist.json` — whitelist entries
- `worlds/<LEVEL_NAME>/` — the actual world data

## Important: Bedrock is UDP, not TCP — and that's not configurable

Bedrock clients (Windows, console, mobile — all of them) use **RakNet**,
which is built on UDP at the protocol level inside the client itself. There
is no server-side setting, env var, or config flag anywhere in the
ecosystem that switches this to TCP. This has real consequences:

- **Cloudflare Tunnel's TCP ingress** (used for the Java server / for
  Crafty's web UI in this repo) does **not** apply to Bedrock traffic at
  all — Bedrock never sends TCP.
- **Cloudflare Tunnel's UDP support** exists, but only for WARP-enrolled
  private network access — every player would need to install Cloudflare
  WARP and be added to your Zero Trust org. Not viable for public players.
- **Cloudflare Spectrum** is the only Cloudflare product that proxies
  public UDP transparently (and explicitly lists Minecraft Bedrock as a
  supported use case) — but it's a paid add-on, not available on the free
  tier.
- **playit.gg** (see [`networking.md`](networking.md)) supports UDP on its
  free tier and is the recommended path here, especially under CGNAT.

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
  if cross-play is the actual goal rather than "I only have the Bedrock
  app handy."
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
the same port, just from the internet instead of the LAN.
