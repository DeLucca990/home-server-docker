# Initial Setup — Prowlarr, Radarr, Sonarr, Bazarr and Jellyfin

First-run configuration guide for the `*arr` stack running on Docker Compose
(LinuxServer.io images), integrated with Jellyfin.

**Scope:** what to configure the first time you open each Web UI. Assumes a working
`compose-medias.yml`, containers up, and all services on the same Compose network.

**Reference versions (Sep 2026):** Radarr 6.4.x · Sonarr 4.0.x · Prowlarr 2.x ·
Bazarr 1.5.x · Jellyfin 12.x · qBittorrent 5.x

---

## Configuration order

Follow this order — each step depends on the previous one.

| # | Step | File |
|---|---|---|
| 0 | Folders, permissions, networking | [00-before-you-start.md](00-before-you-start.md) |
| 1 | qBittorrent — paths, categories, exclusions | [01-qbittorrent.md](01-qbittorrent.md) |
| 2 | Prowlarr part 1 — auth + indexers (+ FlareSolverr) | [02-prowlarr-indexers.md](02-prowlarr-indexers.md) |
| 3 | Radarr | [03-radarr.md](03-radarr.md) |
| 4 | Sonarr | [04-sonarr.md](04-sonarr.md) |
| 5 | Prowlarr part 2 — sync the apps | [05-prowlarr-apps.md](05-prowlarr-apps.md) |
| 6 | Bazarr | [06-bazarr.md](06-bazarr.md) |
| 7 | Jellyfin — libraries + API keys | [07-jellyfin.md](07-jellyfin.md) |
| 8 | Final integrations — notify Jellyfin | [08-integrations.md](08-integrations.md) |

## Afterwards

| Step | File |
|---|---|
| Common problems | [09-troubleshooting.md](09-troubleshooting.md) |
| References | [10-references.md](10-references.md) |

---

## Service map

Containers in the same Compose file resolve each other by **service name**. Always use that —
never `localhost` or `127.0.0.1`.

| Service | Internal hostname | Port |
|---|---|---|
| qBittorrent | `qbittorrent` | 8080 |
| Prowlarr | `prowlarr` | 9696 |
| Radarr | `radarr` | 7878 |
| Sonarr | `sonarr` | 8989 |
| Bazarr | `bazarr` | 6767 |
| Jellyfin | `jellyfin` | 8096 |
| FlareSolverr *([2.3](02-prowlarr-indexers.md#23-flaresolverr-optional))* | `flaresolverr` | 8191 |
