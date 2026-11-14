# 2. Tailscale on the Client

Every device that should reach the server joins the same tailnet, logged in with the **same
identity provider account** used in [1.2](01-tailscale-server.md#12-join-the-tailnet).

---

## 2.1 macOS

Two builds, both fine:

- **Mac App Store** — easiest, auto-updates
- **Standalone** from [tailscale.com/download/mac](https://tailscale.com/download/mac) — needed if
  you want it running as a system daemon (active before login, useful on a machine that reboots
  unattended)

Install, launch, **Log in**, authenticate with the same account. The menu bar icon turns solid
when connected and lists the machines in the tailnet.

The CLI is bundled but not on `PATH` by default:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale status
```

Worth an alias in `~/.zshrc`:

```bash
alias tailscale="/Applications/Tailscale.app/Contents/MacOS/Tailscale"
```

## 2.2 iOS / Android

Install from the App Store or Play Store, open, log in with the same account. Toggle the VPN on
and accept the system VPN profile prompt.

On iOS, turn on **Settings → VPN → Connect On Demand** in the Tailscale app if you want it to
come back automatically after a reboot. Battery impact is small — idle WireGuard sends almost
nothing — but the VPN slot is exclusive, so another always-on VPN cannot run at the same time.

## 2.3 Windows

Installer from [tailscale.com/download/windows](https://tailscale.com/download/windows). Log in
from the tray icon. It installs as a service and reconnects at boot.

## 2.4 Linux

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Add `--accept-routes` if the server is acting as a subnet router — unlike the desktop apps, the
Linux client ignores advertised routes unless told otherwise.

## 2.5 Verify the connection

From the client:

```bash
tailscale status          # the server should be listed
ping home-server          # MagicDNS resolving the short name
tailscale netcheck        # NAT/DERP diagnostics if something is off
```

Then open `http://home-server:8096` in a browser. If the name does not resolve but the
`100.x.y.z` address works, MagicDNS is the problem — check it is enabled in the admin console and
that the client is not being overridden by another VPN or a custom DNS setting.

## 2.6 Using the services

| Service | URL |
|---|---|
| Jellyfin | `http://home-server:8096` |
| qBittorrent | `http://home-server:8080` |
| Prowlarr | `http://home-server:9696` |
| Radarr | `http://home-server:7878` |
| Sonarr | `http://home-server:8989` |
| Bazarr | `http://home-server:6767` |
| Portainer | `https://home-server:9443` |
| Uptime Kuma | `http://home-server:3001` |

These work identically at home and away through the encrypted tailnet. At home, devices that do
not run Tailscale can use the server's LAN address directly.

If you set up HTTPS in
[1.6](01-tailscale-server.md#16-https-on-the-existing-ports-optional-but-recommended), the port
numbers stay exactly the same and only the scheme and the hostname change — the certificate is
issued for the full name, so the short name no longer works:

```text
https://home-server.<tailnet>.ts.net:8096     # Jellyfin
https://home-server.<tailnet>.ts.net:7878     # Radarr
```

**Jellyfin apps** (mobile, TV, desktop): add the server as `http://home-server:8096`, or as
`https://home-server.<tailnet>.ts.net:8096` once it is served over TLS. Note that a TV or
streaming stick that cannot run Tailscale can use the server's LAN address directly.

## 2.7 Subnet routes and exit node

Only relevant if you enabled them on the server ([1.7](01-tailscale-server.md#17-optional--subnet-router-for-the-whole-lan)
and [1.8](01-tailscale-server.md#18-optional--exit-node)).

| Client | Subnet routes | Exit node |
|---|---|---|
| macOS / Windows | menu → **Use Tailscale Subnets** | menu → **Exit Node** → `home-server` |
| iOS / Android | app → **Settings → Use Tailscale Subnets** | app → **Exit Node** |
| Linux | `sudo tailscale up --accept-routes` | `sudo tailscale up --exit-node=home-server` |

While an exit node is selected, *all* of that device's traffic goes through your home connection
— including the part you may not want to backhaul. It is a toggle, not a permanent setting.

## 2.8 Common problems

| Symptom | Likely cause | Fix |
|---|---|---|
| `home-server` does not resolve, IP works | MagicDNS off, or another VPN/DNS overriding it | enable MagicDNS in the admin console; disconnect other VPNs |
| Nothing connects at all | device logged into a different tailnet | check the account in the client; the admin console **Machines** list must show both devices |
| Worked for months, suddenly offline | node key expired | [1.3](01-tailscale-server.md#13-stop-the-node-key-from-expiring) — disable key expiry on the server |
| Connects but slow / high latency | relayed through DERP instead of direct | `tailscale netcheck` on both ends; usually a restrictive NAT or a blocked UDP port |
| LAN devices unreachable through the server | routes advertised but not approved | approve in the admin console, and enable "accept routes" on the client |
| Certificate warning on `https://home-server:8096` | the cert is issued for the full `ts.net` name | use `https://home-server.<tailnet>.ts.net:8096` |
| `http://home-server:8096` broke after enabling HTTPS | that port now terminates TLS over the tailnet | switch the bookmark to `https://` + the full name |
| Jellyfin logs every user as `127.0.0.1` | `tailscale serve` is the proxy | add `127.0.0.1` to Jellyfin's Known proxies ([1.6](01-tailscale-server.md#16-https-on-the-existing-ports-optional-but-recommended)) |
| Can reach the server, but not from a TV app | the TV cannot run Tailscale | use the server's LAN address, or update `&lan-address` in `compose-medias.yml` |
| Two VPNs fighting on mobile | the OS allows one VPN profile at a time | disconnect the other one |

---

Previous: [1. Tailscale on the server](01-tailscale-server.md) · Back to [expose index](README.md)
