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

Then open `https://home-server:9443` in a browser — Portainer listens on the tailnet directly, so
it answers as soon as the connection works (accept its self-signed certificate). The other
services answer only once the server serves them
([1.6](01-tailscale-server.md#16-serve-the-services-over-https)).

If `ping home-server` fails but pinging the `100.x.y.z` address works, MagicDNS is the problem —
check it is enabled in the admin console and that the client is not being overridden by another
VPN or a custom DNS setting.

## 2.6 Using the services

With the services served over HTTPS on the server
([1.6](01-tailscale-server.md#16-serve-the-services-over-https)):

| Service | URL |
|---|---|
| Jellyfin | `https://home-server.<tailnet>.ts.net:8096` |
| qBittorrent | `https://home-server.<tailnet>.ts.net:8080` |
| Prowlarr | `https://home-server.<tailnet>.ts.net:9696` |
| Radarr | `https://home-server.<tailnet>.ts.net:7878` |
| Sonarr | `https://home-server.<tailnet>.ts.net:8989` |
| Bazarr | `https://home-server.<tailnet>.ts.net:6767` |
| Portainer | `https://home-server:9443` |
| Uptime Kuma | `https://home-server.<tailnet>.ts.net:3001` |

Use the full name: the certificate is issued for `home-server.<tailnet>.ts.net`, so the short
name gives a certificate warning. Portainer is the exception — it is not behind `serve` and
answers on the short name with its own certificate.

These work identically at home and away through the encrypted tailnet. At home, devices that do
not run Tailscale can use the server's LAN address directly (`http://SERVER_IP:<port>`); with the
subnet router from [1.7](01-tailscale-server.md#17-optional--subnet-router-for-the-whole-lan),
so can tailnet devices anywhere.

**Jellyfin apps** (mobile, TV, desktop): add the server as
`https://home-server.<tailnet>.ts.net:8096`. A TV or streaming stick that cannot run Tailscale
can use the server's LAN address directly.

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
| `http://home-server:<port>` gets no answer | Docker listens only on loopback and the LAN address; only `tailscale serve` answers on the tailnet | run the serve commands from [1.6](01-tailscale-server.md#16-serve-the-services-over-https) and use `https://` + the full name |
| Jellyfin logs every user as `127.0.0.1` | `tailscale serve` is the proxy | add `127.0.0.1` to Jellyfin's Known proxies ([1.6](01-tailscale-server.md#16-serve-the-services-over-https)) |
| Can reach the server, but not from a TV app | the TV cannot run Tailscale | use the server's LAN address, and check that it matches `LAN_IP` in `.env` ([fixed LAN IP](../network/README.md)) |
| Two VPNs fighting on mobile | the OS allows one VPN profile at a time | disconnect the other one |

---

Previous: [1. Tailscale on the server](01-tailscale-server.md) · Back to [expose index](README.md)
