# 9. Common Problems

| Symptom | Likely cause | Fix |
|---|---|---|
| Radarr/Sonarr cannot reach qBittorrent | used `localhost` instead of the container name | Host = `qbittorrent` |
| Connection refused despite the right hostname | qBittorrent host header validation | Web UI → *Server domains* → `*` or `qbittorrent` |
| Imports become copies, disk fills up | download and library on different mounts | Radarr/Sonarr must mount the whole `/home/ubuntu/media` as `/media` — see [0.1](00-before-you-start.md#01-folder-structure) |
| "Download client is reporting files are in X but this directory does not appear to exist" | path mismatch between containers | the path string must be identical in both — qBittorrent's `/media/torrents` is Radarr's `/media/torrents` |
| Indexer missing in Radarr after sync | incompatible categories | check the indexer's categories in Prowlarr |
| Indexer test fails with a Cloudflare / `403` / `503` error | indexer behind Cloudflare | route it through FlareSolverr — [2.3](02-prowlarr-indexers.md#23-flaresolverr-optional) |
| FlareSolverr configured but the indexer still fails on Cloudflare | indexer missing the `flaresolverr` tag | the indexer and the proxy must share the same tag — [2.3.3](02-prowlarr-indexers.md#233-tag-the-indexers-that-need-it) |
| Proxy test fails: cannot connect to `flaresolverr` | container moved to `compose-utils.yml`, or not running | it must be in `compose-medias.yml`; check `docker logs flaresolverr` |
| FlareSolverr times out or cannot resolve the site | ISP DNS blocking the domain | confirm the `dns:` entries (`1.1.1.1`, `8.8.8.8`) are in the `flaresolverr` service and recreate it with `up -d` |
| FlareSolverr times out or "Challenge not detected" after working before | Cloudflare changed its challenge | pull the latest image; raise Request Timeout; interactive captchas are not solvable |
| Indexer disappeared from Radarr/Sonarr after tagging it | the app in Prowlarr has tags of its own | remove the app's tags, or add `flaresolverr` to the app too |
| Bazarr finds the series but downloads nothing | Minimum Score too high or too few providers | lower the score, add providers |
| Bazarr throws a path error | Path Mapping filled in unnecessarily | clear the Path Mappings |
| Excluded extensions still downloading | known 5.x bugs with magnet/RSS/API-added torrents | verify on the Content tab; see [1.3](01-qbittorrent.md#13-excluded-file-names) |
| CPU pinned at 100% during playback | an x265 file falling back to software transcoding | Custom Format `x265 (HD)` scored -10000 |
| New media never appears in Jellyfin | missing Emby/Jellyfin connection | [step 8](08-integrations.md) |
| Torrent disappears before seeding completes | qBittorrent set to remove | seeding limit action = `Pause torrent` |
| Uptime Kuma cannot resolve `jellyfin` | separate Compose stacks, separate networks | monitor by `http://SERVER_IP:8096` |

---

Previous: [8. Final integrations](08-integrations.md) · Next: [10. References](10-references.md)
