# Fixed LAN IP — DHCP Reservation

The Web UIs are published on the server's LAN address, set as `LAN_IP` in `.env` (see
[Host binding](../../README.md#host-binding)). Docker can only bind to an address the server
actually has, so if the router hands it a different one after a reboot or a lease renewal, every
container with a LAN port fails to start:

```text
Error response from daemon: ... listen tcp4 192.168.1.50:8096: bind: cannot assign requested address
```

A DHCP reservation prevents that: the router always hands the server the same address, and
nothing has to change on the server itself. The same fixed address is also what the router's
port forward for qBittorrent ([1.6](../media/01-qbittorrent.md#16-incoming-connections-port))
points at.

---

## 1. Collect the server's network details

On the server:

```bash
ip route get 1.1.1.1
```

```text
1.1.1.1 via 192.168.1.1 dev enp3s0 src 192.168.1.50 uid 1000
```

| Field | Meaning | Example |
|---|---|---|
| `via` | the router's address — where its admin page lives | `192.168.1.1` |
| `dev` | the network interface the server uses | `enp3s0` |
| `src` | the server's current LAN IP | `192.168.1.50` |

Then the MAC address of that interface (replace `enp3s0` with the `dev` above):

```bash
cat /sys/class/net/enp3s0/address
```

```text
a4:bb:6d:12:34:56
```

If the server has both Ethernet and Wi-Fi, each has its own MAC — reserve the one shown as `dev`.

## 2. Pick the address

Reserving the **current** address (`src`) is the easy path: nothing changes on the server and no
connection drops. Pick another one only if you have a reason to, and check in the router's list
of connected devices that no other device is using it.

## 3. Create the reservation on the router

1. On any device on the LAN, open `http://<via>` — `http://192.168.1.1` in the example.
2. Log in. ISP-supplied routers usually print the admin credentials on a label under the device.
3. Find the reservation screen. Its name and location vary by vendor; look under **LAN**,
   **DHCP** or **Network** for something like:
   - *Address Reservation* / *DHCP Reservation*
   - *Static DHCP* / *DHCP Static Lease*
   - *IP & MAC Binding*
   - on Portuguese firmware: *Reserva de IP*, *Reserva de endereço*, *DHCP estático*

   Some routers also have a "reserve" button right in the connected-devices list.
4. Add an entry with the MAC from step 1 and the IP from step 2. If there is a name field, it is
   free text — `home-server` works.
5. Save, and apply or reboot the router if it asks.

If the router rejects the address, check the DHCP range on the same page: some routers only
accept reservations inside the range, others only outside it.

## 4. Make the server pick it up

If you reserved the current address, there is nothing to do — the next lease renewal simply
confirms it.

If you reserved a different one, renew the lease (or reboot):

```bash
sudo networkctl renew enp3s0
```

Over SSH the session drops when the address changes; reconnect to the new one.

## 5. Verify

```bash
ip route get 1.1.1.1    # src must be the reserved address
```

Then reboot the server once and check again, to confirm the reservation survives a restart.

## 6. Point the stacks at it

In `.env`, next to the Compose files (copy it from `.env.example` if it does not exist yet):

```bash
LAN_IP=192.168.1.50
```

Recreate both stacks so the ports bind to it:

```bash
docker compose -f compose-medias.yml up -d
docker compose -f compose-utils.yml up -d
```

## 7. The server keeps getting another address

Ubuntu Server's network backend (systemd-networkd, configured through netplan) identifies itself
to the DHCP server with a generated client ID instead of the MAC address. Routers that match
reservations on the client ID never recognise the server, and keep handing it a pool address.

Tell netplan to identify with the MAC. Use the same interface key as the existing file in
`/etc/netplan/` (usually the interface name):

```bash
sudo tee /etc/netplan/90-dhcp-identifier.yaml >/dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp-identifier: mac
EOF
sudo chmod 600 /etc/netplan/90-dhcp-identifier.yaml
sudo netplan try
```

`netplan try` applies the change and rolls it back after 120 seconds unless you confirm with
Enter — a safety net when working over SSH. Then go back to [5](#5-verify).

## 8. The router has no reservation option

Some locked-down ISP routers do not offer reservations. In that case, give the server a static
address in netplan instead, **outside** the router's DHCP range so the router never hands it to
another device. Replace the interface's `dhcp4: true` in the existing file under `/etc/netplan/`
with:

```yaml
network:
  version: 2
  ethernets:
    enp3s0:
      dhcp4: false
      addresses: [192.168.1.200/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1]
```

`/24` matches the usual `255.255.255.0` mask — confirm with `ip -4 addr show enp3s0`. `via` and
the nameserver are the router's address from step 1. Apply with `sudo netplan try`, then follow
[5](#5-verify) and [6](#6-point-the-stacks-at-it) with the new address.

---

Back to [README](../../README.md)
