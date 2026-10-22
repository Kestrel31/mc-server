# Networking: going public behind CGNAT

This network is behind **Carrier-Grade NAT (CGNAT)** — the ISP shares its
"public" IP across many customers, so there is no port that can be opened
to the internet from the router. Port forwarding and dynamic DNS pointed
at the home IP are both non-starters: there's no public IP to point at or
forward through in the first place.

That rules out the "classic home server" model (DNS record → home IP →
router port forward) entirely, regardless of which DNS provider is used.
Every viable option below works by having something on this network dial
**out** to a public rendezvous point instead of accepting inbound
connections directly.

## Options considered

| Option | Cost | UDP support | Hides home IP | Setup effort | Notes |
|---|---|---|---|---|---|
| **playit.gg** ✅ chosen | Free | Yes | Yes | Low | Purpose-built for game servers; official Docker image; outbound-only agent |
| Cloudflare Tunnel | Free | WARP-client only | Yes | Medium | TCP ingress works great for Java / for a web panel's UI; **not usable for public Bedrock UDP** without every player running WARP |
| Cloudflare Spectrum | Paid add-on | Yes | Yes | Medium | The "proper" Cloudflare answer for public UDP, including Bedrock — not available on free tier |
| frp (self-hosted) | ~$5/mo VPS | Yes | Yes | Medium-high | Fully self-owned, free aside from VPS; more setup than playit.gg for the same outcome |
| ngrok | Paid for UDP | Yes (paid only) | Yes | Low | Free tier is TCP/HTTP only |
| Tailscale / ZeroTier | Free | Yes | N/A (private mesh) | Low | Great for a small trusted friend group who'll install the client; **not** suited for "anyone can join via an address" public access |

## Why playit.gg

- Free tier supports UDP, which Bedrock requires (see
  [`bedrock.md`](bedrock.md) for why that's non-negotiable) and which Java
  also benefits from avoiding the WARP-client requirement.
- No port forwarding, no static IP, no router config — works under CGNAT
  by design, since the agent only makes outbound connections.
- Official Docker image (`ghcr.io/playit-cloud/playit-agent`), included
  directly in `compose/docker-compose.java.playit.yml` and
  `compose/docker-compose.bedrock.playit.yml`.
- Purpose-built for game servers (Minecraft Bedrock specifically called
  out), rather than a general-purpose tunnel product repurposed for this.

## Setup

1. Create a tunnel agent at [playit.gg](https://playit.gg) → copy the
   generated `SECRET_KEY`.
2. Put it in `.env` as `PLAYIT_SECRET_KEY` (never commit this — it's a
   credential tied to your account; rotate it from the dashboard if it's
   ever exposed).
3. Use one of the `*.playit.yml` compose files instead of the plain ones:
   ```bash
   docker compose -f compose/docker-compose.java.playit.yml up -d
   # or
   docker compose -f compose/docker-compose.bedrock.playit.yml up -d
   ```
4. In the playit.gg dashboard, set the tunnel's local address to the
   Docker service name and port — `mc:25565` for Java or `bedrock:19132`
   for Bedrock — since `playit` and the game server share the same
   Compose network, Docker's internal DNS resolves the service name.
   (Do **not** use `network_mode: host` — Docker Desktop on Windows only
   has experimental/inconsistent support for it; bridge networking with
   the service name works reliably everywhere and is what these compose
   files use.)
5. Give players the hostname playit.gg generates (e.g.
   `yourserver.joinmc.link`) — they put it straight into the client, no
   extra software needed on their end.

## A web panel's UI, if one is ever reintroduced

If a web panel is ever added back purely for console/backups/file-browsing
(not as the process owner — see [`decisions.md`](decisions.md)), its own
web port is plain HTTPS and is a good fit for a normal **Cloudflare
Tunnel** public hostname — unlike the game port, this doesn't hit the UDP
limitation. Layer **Cloudflare Access** in front of it so the panel's own
login page isn't reachable by anyone without first passing a separate
Cloudflare identity check.

```yaml
# cloudflared/config.yml
tunnel: <TUNNEL-ID>
credentials-file: /etc/cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: panel.yourdomain.com
    service: https://filebrowser:80
  - service: http_status:404
```

```bash
cloudflared tunnel login
cloudflared tunnel create minecraft
cloudflared tunnel route dns minecraft panel.yourdomain.com
```
