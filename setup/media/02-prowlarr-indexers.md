# 2. Prowlarr — Part 1 (Authentication and Indexers)

`http://SERVER_IP:9696`

Prerequisite: [1. qBittorrent](01-qbittorrent.md)
Part 2 (syncing the apps) comes after Radarr and Sonarr exist: [5. Prowlarr — Apps](05-prowlarr-apps.md)

---

## 2.1 First access

Authentication is mandatory in modern `*arr` apps. The first screen asks for method and
credentials:

- **Authentication Method**: `Forms (Login Page)` — recommended
- **Authentication Required**: `Enabled` (or `Disabled for Local Addresses` for LAN-only access)

Then grab the API key at `Settings → General → Security → API Key`.

## 2.2 Add indexers

`Indexers → +`

Every available indexer (usenet and torrent) is listed and you can type to filter. If the one you
want is not there, add "Generic Newznab" (usenet) or "Generic Torznab" (torrents).

Fields that matter on first setup:

- **Name** — when synced to your apps, `(Prowlarr)` is appended to it
- **Sync Profile** — leave as `Standard` for now
- **URL** — blank uses the indexer's default URL
- **Credentials** — API key / RSS key / cookie, depending on the indexer
- **Tags** — useful to route indexers to specific apps or to a proxy (see 2.3)
- *(Advanced)* **Query Limit / Grab Limit** — fill these in if the indexer has a daily quota.
  Once the grab limit is reached, further queries throw an unhandled exception in the `*arr` apps.

Hit **Test** on each one. Only save on a green check.

### Two public torrent indexers to start with

Both are public trackers: type the name in the filter, leave **Credentials** and **URL** blank,
and hit Test.

| Indexer | Covers | Notes |
|---|---|---|
| **YTS** | Movies only | Small x264/x265 encodes, one release per quality. Being movies-only, it syncs to Radarr and never reaches Sonarr — see the category note in [5. Prowlarr — Apps](05-prowlarr-apps.md). |
| **The Pirate Bay** | Movies + TV | General-purpose, large catalogue, no account needed. Syncs to both Radarr and Sonarr. |

Two caveats before you rely on them:

- **YTS is low-bitrate by design.** The YIFY/YTS group is part of TRaSH's `LQ` (Low Quality
  Groups) Custom Format, so if you import `LQ` with a negative score in
  [Radarr 3.3](03-radarr.md#33-profiles), YTS releases get rejected and the indexer becomes dead
  weight. Pick one: either skip the `LQ` format, or treat YTS as a last-resort fallback.
- **The Pirate Bay is unmoderated.** Fake releases carrying an executable are routine there, which
  is exactly what the excluded file names in
  [1.3](01-qbittorrent.md#13-excluded-file-names) defend against. Pair it with a Sync Profile
  setting `Minimum Seeders: 3` ([5.2](05-prowlarr-apps.md#52-sync-profiles)) to filter out dead
  and fake torrents.

If either test fails with a Cloudflare error, that is what FlareSolverr below is for.

## 2.3 FlareSolverr (optional)

Some indexers sit behind Cloudflare's anti-bot page ("Checking your browser…"). Prowlarr cannot
pass that challenge on its own, so the indexer test fails. [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr)
is a small proxy that opens the page in a headless Chromium, solves the challenge and hands the
cookies back to Prowlarr.

The container is already part of `compose-medias.yml` and starts with the rest of the stack
(the headless browser costs ~200–400 MB of RAM). What is optional is wiring it into Prowlarr —
do that only for indexers that actually need it. Typical signs:

- the indexer test fails with a message mentioning Cloudflare, or a `403` / `503` error
- Prowlarr's logs (`System → Logs`) show `Cloudflare protection detected`
- the indexer's site opens in a normal browser only after the "Checking your browser" screen

### 2.3.1 The container

It lives in **`compose-medias.yml`**, in the same file as Prowlarr, so that the `flaresolverr`
hostname resolves on the Compose network (see
[0.3](00-before-you-start.md#03-container-to-container-communication)). Moving it to
`compose-utils.yml` would break that.

```yaml
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
      - TZ=America/Sao_Paulo
    dns:
      - 1.1.1.1
      - 8.8.8.8
    ports:
      - 8191:8191
    restart: unless-stopped
```

`dns` makes the container resolve names through Cloudflare (`1.1.1.1`) with Google (`8.8.8.8`)
as fallback, instead of the host's resolver — which is usually the ISP's DNS via the router.
ISP resolvers commonly block or hijack torrent sites, and a blocked lookup surfaces as a
FlareSolverr timeout or a generic connection error rather than a clear DNS failure.

The port is published on `127.0.0.1` only. Prowlarr reaches FlareSolverr over the Compose network
and does not need it published at all; the localhost binding exists just so you can test it from
the server itself. FlareSolverr has **no authentication**, so do not expose it to the LAN or the
tailnet.

It comes up with `docker compose -f compose-medias.yml up -d`. If the rest of the stack is already
running, starting just this one is enough:

```bash
docker compose -f compose-medias.yml up -d flaresolverr
```

Check that it is up:

```bash
docker logs flaresolverr        # should end with "Serving on http://0.0.0.0:8191"
curl http://127.0.0.1:8191/     # {"msg": "FlareSolverr is ready!", ...}
docker exec flaresolverr cat /etc/resolv.conf   # nameserver 127.0.0.11 is expected (see below)
```

`resolv.conf` showing `127.0.0.11` is normal: that is Docker's embedded resolver, which answers
service names like `prowlarr` and forwards everything else to `1.1.1.1` / `8.8.8.8`.

### 2.3.2 Register it in Prowlarr

`Settings → Indexers → Indexer Proxies → + → FlareSolverr`

| Field | Value |
|---|---|
| **Name** | `FlareSolverr` |
| **Tags** | `flaresolverr` |
| **Host** | `http://flaresolverr:8191/` |
| **Request Timeout** | `60` (default; raise it if solves time out on a slow server) |

Hit **Test**, then **Save**.

### 2.3.3 Tag the indexers that need it

The proxy is **only used by indexers that carry the same tag**. A FlareSolverr proxy whose tag
matches no indexer does nothing.

`Indexers →` click the indexer `→ Tags → flaresolverr → Test → Save`

Tag only the indexers that are actually behind Cloudflare. Every request through FlareSolverr
spins up a browser page, which is slower and heavier than a plain HTTP request, and Prowlarr only
hands a request to it when it detects a Cloudflare challenge anyway.

> **Tags also filter app sync.** In Prowlarr, if an app in `Settings → Apps` has tags, only
> indexers with a matching tag are synced to it. The apps in [step 5](05-prowlarr-apps.md) are
> added without tags, so the `flaresolverr` tag does not affect syncing — keep it that way, or
> add the same tag to the app as well.

### 2.3.4 Limitations

- FlareSolverr solves Cloudflare's JavaScript challenge. It **cannot** solve interactive captchas
  (Turnstile checkbox, hCaptcha). If an indexer shows one of those, FlareSolverr will not help —
  look for a mirror or a different indexer.
- Cloudflare changes its challenge regularly, and FlareSolverr breaks from time to time until a
  new release ships. If a previously working indexer starts failing, update first:
  `docker compose -f compose-medias.yml pull flaresolverr && docker compose -f compose-medias.yml up -d flaresolverr`.

### 2.3.5 Removing it

If no indexer needs it and you want the RAM back: remove the tag from the indexers, delete the
proxy in Prowlarr, delete the `flaresolverr` block from `compose-medias.yml` and run:

```bash
docker compose -f compose-medias.yml up -d --remove-orphans
```

## 2.4 Download clients in Prowlarr

Only needed if you plan to search from Prowlarr's own UI. Download clients configured in Prowlarr
are used exclusively for in-app searches there and are never synced to the apps — searches
initiated from Radarr/Sonarr use the clients configured in those apps.

If you want it: `Settings → Download Clients → + → qBittorrent` → Host `qbittorrent`,
Port `8080`, your credentials.

---

Previous: [1. qBittorrent](01-qbittorrent.md) · Next: [3. Radarr](03-radarr.md)
