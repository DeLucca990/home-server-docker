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

Indexers behind Cloudflare need a solver. **It is not part of `compose-medias.yml`** — the
commented-out block was removed deliberately (it costs ~200–400 MB of RAM). Add it back only if
an indexer you actually use requires it:

```yaml
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
      - TZ=America/Sao_Paulo
    ports:
      - 8191:8191
    restart: unless-stopped
```

Then in `Settings → Indexer Proxies → + → FlareSolverr`:

- **Host**: `http://flaresolverr:8191`
- **Tags**: `flaresolverr`

Apply the `flaresolverr` tag to the indexers that need it.

## 2.4 Download clients in Prowlarr

Only needed if you plan to search from Prowlarr's own UI. Download clients configured in Prowlarr
are used exclusively for in-app searches there and are never synced to the apps — searches
initiated from Radarr/Sonarr use the clients configured in those apps.

If you want it: `Settings → Download Clients → + → qBittorrent` → Host `qbittorrent`,
Port `8080`, your credentials.

---

Previous: [1. qBittorrent](01-qbittorrent.md) · Next: [3. Radarr](03-radarr.md)
