# 1. Tailscale on the Server

Ubuntu Server, alongside the running Docker stacks. The media stack binds its published ports to
loopback so it can coexist with Tailscale Serve.

---

## 1.1 Install

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

The script detects the distribution, adds Tailscale's apt repository and installs
`tailscaled` as a systemd service. Confirm it is running:

```bash
systemctl status tailscaled --no-pager
```

## 1.2 Join the tailnet

```bash
sudo tailscale up --ssh --hostname=home-server
```

This prints a login URL — open it on any machine and authenticate with the identity provider you
want to own the tailnet (Google, GitHub, Microsoft, email).

| Flag | Why |
|---|---|
| `--hostname=home-server` | Fixes the MagicDNS name instead of inheriting whatever the OS hostname is. Lowercase, no dots. |
| `--ssh` | Enables **Tailscale SSH**: you can `ssh ubuntu@home-server` from any tailnet device with no keys to manage, and access is controlled by tailnet policy. Drop it if you prefer plain OpenSSH. |

For a fully unattended install, generate an auth key in the admin console
(`Settings → Keys → Generate auth key`) and pass `--auth-key=tskey-auth-...` instead of
authenticating in a browser.

Verify:

```bash
tailscale status
tailscale ip -4
```

## 1.3 Stop the node key from expiring

By default a node key expires after 180 days and the machine silently drops off the tailnet —
which, on a headless server you are not sitting in front of, means losing remote access at the
worst possible moment.

In the admin console → **Machines** → `home-server` → `⋯` → **Disable key expiry**.

## 1.4 MagicDNS names

MagicDNS is on by default for new tailnets (admin console → **DNS** if you need to check). Once
it is on, the server answers to:

```text
home-server                          # short name, from any tailnet device
home-server.<tailnet-name>.ts.net    # fully qualified
100.x.y.z                            # stable Tailscale IP, never changes
```

Find your tailnet name at the top of the DNS page in the admin console — it looks like
`tail1a2b3c.ts.net` unless you renamed it.

## 1.5 Reaching the Docker services

The media stack publishes its ports on both loopback and the LAN address. Use Tailscale Serve to
proxy the loopback listener to the tailnet address:

| Service | Over Tailscale |
|---|---|
| Jellyfin | `http://home-server:8096` |
| qBittorrent | `http://home-server:8080` |
| Prowlarr | `http://home-server:9696` |
| Radarr | `http://home-server:7878` |
| Sonarr | `http://home-server:8989` |
| Bazarr | `http://home-server:6767` |
| Portainer | `https://home-server:9443` |
| Uptime Kuma | `http://home-server:3001` |

These are plain HTTP over the encrypted tunnel, which is fine for a private tailnet. Configure
the proxy commands in [1.6](#16-https-on-the-existing-ports-optional-but-recommended).

> The container-to-container hostnames from the setup guide (`radarr`, `qbittorrent`, …) are
> unrelated to this and keep working exactly as before — they are internal to the Compose
> network. Tailscale only changes how *you* reach the host.

## 1.6 HTTPS on the existing ports (optional but recommended)

`tailscale serve` puts a TLS terminator in front of a local port using a real Let's Encrypt
certificate issued for your `ts.net` name. Browsers stop complaining, and clients that insist on
HTTPS work properly.

Left to itself, `serve` takes over port **443**, which would make the bare
`https://home-server.<tailnet-name>.ts.net` mean Jellyfin. Pass `--https=<port>` explicitly
instead: each service keeps the port number it already has, the URL pattern stays uniform, and
443 stays free for whatever you may want at the root later.

First enable **HTTPS Certificates** in the admin console → **DNS**. Then one command per service,
serving each port onto itself:

```bash
sudo tailscale serve --bg --https=8096 http://127.0.0.1:8096   # Jellyfin
sudo tailscale serve --bg --https=8080 http://127.0.0.1:8080   # qBittorrent
sudo tailscale serve --bg --https=9696 http://127.0.0.1:9696   # Prowlarr
sudo tailscale serve --bg --https=7878 http://127.0.0.1:7878   # Radarr
sudo tailscale serve --bg --https=8989 http://127.0.0.1:8989   # Sonarr
sudo tailscale serve --bg --https=6767 http://127.0.0.1:6767   # Bazarr
sudo tailscale serve --bg --https=3001 http://127.0.0.1:3001   # Uptime Kuma
```

Result — same ports as before, now over TLS:

| Service | Over Tailscale |
|---|---|
| Jellyfin | `https://home-server.<tailnet-name>.ts.net:8096` |
| qBittorrent | `https://home-server.<tailnet-name>.ts.net:8080` |
| Prowlarr | `https://home-server.<tailnet-name>.ts.net:9696` |
| Radarr | `https://home-server.<tailnet-name>.ts.net:7878` |
| Sonarr | `https://home-server.<tailnet-name>.ts.net:8989` |
| Bazarr | `https://home-server.<tailnet-name>.ts.net:6767` |
| Uptime Kuma | `https://home-server.<tailnet-name>.ts.net:3001` |

Portainer is left out on purpose: it already terminates its own TLS on `9443`, so putting
`serve` in front would be TLS inside TLS. Reach it at `https://home-server:9443` as before, or
proxy its HTTP port if you prefer one consistent scheme.

Three consequences worth knowing before you run these:

- **The full name is now mandatory.** The certificate is issued for
  `home-server.<tailnet-name>.ts.net`, so the MagicDNS short name gives a certificate mismatch
  in the browser. `https://home-server:8096` will not do — use the FQDN.
- **That port speaks TLS over the tailnet from now on.** Plain `http://home-server:8096` stops
  working once 8096 is served; it is one port, one protocol.
- **LAN access remains direct.** The LAN listener is configured by `&lan-address` in the
  `x-lan-address` configuration at the top of `compose-medias.yml`.

Managing it:

```bash
tailscale serve status              # what is currently proxied
sudo tailscale serve --https=8096 off   # remove one
tailscale serve reset               # tear it all down
```

**Jellyfin gotcha:** with `serve` in front, every request arrives from `127.0.0.1`, so Jellyfin
logs one client IP for everyone and may misjudge what counts as a local connection. In
`Dashboard → Networking`, add `127.0.0.1` to **Known proxies**, and set the **Published server
URL** to `https://home-server.<tailnet-name>.ts.net:8096` so clients get sent to the right place.

## 1.7 Optional — subnet router for the whole LAN

Only needed if you want tailnet devices to reach *other* machines on the home network (the
router's admin page, a printer, a NAS), not just this server.

Enable forwarding:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Advertise the LAN subnet (adjust to match yours — check with `ip route`):

```bash
sudo tailscale up --ssh --hostname=home-server --advertise-routes=192.168.1.0/24
```

Then approve the route in the admin console → **Machines** → `home-server` → **Edit route
settings**. Routes are not active until approved.

Clients also have to accept subnet routes — see
[2.7](02-tailscale-client.md#27-subnet-routes-and-exit-node).

## 1.8 Optional — exit node

Makes the server act as a VPN gateway so a phone on public Wi-Fi can route all its traffic
through your home connection:

```bash
sudo tailscale up --ssh --hostname=home-server --advertise-exit-node
```

Approve it in the admin console the same way, then pick it from the client's menu. Note that
every flag has to be repeated on each `tailscale up` — the command replaces the previous
configuration rather than adding to it.

## 1.9 Funnel — public exposure

`tailscale funnel` takes a service served by `tailscale serve` and publishes it on the **public
internet** at your `ts.net` name, reachable by anyone.

Do not do this for qBittorrent, Portainer, or any `*arr` app: those have weak or no
authentication in front of a torrent client and full filesystem access. Even for Jellyfin it
means exposing a media server, on your home IP, to the open internet — with all the account
hygiene, patching and legal exposure that implies.

Funnel also breaks the port scheme from [1.6](#16-https-on-the-existing-ports-optional-but-recommended):
it only accepts **443, 8443 and 10000**, so a funnelled service cannot keep its own port number.

If you do it anyway, enable Funnel for the node in the admin console ACL policy first, then:

```bash
sudo tailscale funnel --bg --https=443 http://127.0.0.1:8096
tailscale funnel status
tailscale funnel reset
```

## 1.10 Hardening

### Restrict who reaches what (ACLs)

The default tailnet policy lets every device talk to every other device on every port. If more
than one person joins the tailnet, tighten it in the admin console → **Access controls**. A
minimal example that gives a family member Jellyfin only, while you keep full access:

```json
{
  "tagOwners": {
    "tag:media": ["autogroup:admin"]
  },
  "acls": [
    {
      "action": "accept",
      "src":    ["autogroup:admin"],
      "dst":    ["tag:media:*"]
    },
    {
      "action": "accept",
      "src":    ["family@example.com"],
      "dst":    ["tag:media:8096"]
    }
  ]
}
```

Apply the tag to the node with `sudo tailscale up --advertise-tags=tag:media ...` (plus the other
flags you already use). Tagged devices are owned by the tailnet rather than by a user, and their
keys do not expire — which also covers [1.3](#13-stop-the-node-key-from-expiring).

### Close the ports to everything else (optional)

With Tailscale working, the services no longer need to listen to the wider network. UFW can
allow the tailnet interface and nothing else:

```bash
sudo ufw allow in on tailscale0
sudo ufw allow 22/tcp            # keep a way in that does not depend on Tailscale
sudo ufw enable
```

> ⚠️ Two warnings. First, this also cuts off **LAN** access to the Web UIs — devices at home will
> have to use Tailscale too. Second, enabling UFW over a remote session can lock you out; confirm
> `tailscale status` is healthy and that you have console or LAN access before running
> `ufw enable`.
>
> Note that Docker publishes ports by writing its own iptables rules, which UFW does not manage.
> Verify from outside the tailnet that a port is genuinely closed rather than assuming it.
>

---

Next: [2. Tailscale on the client](02-tailscale-client.md) · Back to [expose index](README.md)
