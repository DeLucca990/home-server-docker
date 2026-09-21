# Remote Access — Tailscale

How to reach the server from outside the house without opening a single port on the router.

---

## What this is

Tailscale builds a private WireGuard mesh between your devices. Every device that joins your
tailnet gets a stable `100.x.y.z` address and a DNS name, and traffic between them is encrypted
end-to-end and punches through NAT on its own. The router keeps every port closed.

For this stack that means Jellyfin, Radarr, Sonarr, Prowlarr, Bazarr, qBittorrent, Portainer and
Uptime Kuma become reachable from a laptop or phone anywhere, with **no port forwarding, no
dynamic DNS and no reverse proxy**.

## What this is not

Tailscale is *private* access, not public exposure. Only devices logged into your tailnet can
see the server — someone with the URL and no Tailscale account gets nothing.

If you genuinely want a service on the public internet, that is **Tailscale Funnel**, covered
with its caveats in [1.9](01-tailscale-server.md#19-funnel--public-exposure). For a media stack
the answer is almost always no.

## Order

| # | Step | File |
|---|---|---|
| 1 | Server — install, MagicDNS, HTTPS, hardening | [01-tailscale-server.md](01-tailscale-server.md) |
| 2 | Client — macOS, iOS/Android, Windows, Linux | [02-tailscale-client.md](02-tailscale-client.md) |

Do the server first: the client has nothing to connect to until the server is in the tailnet.

## Plan

The free plan covers this use case comfortably — up to 6 users, with subnet
routing, exit nodes, MagicDNS and HTTPS certificates all included.

---

Back to [media stack setup](../media/README.md)
