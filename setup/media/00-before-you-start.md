# 0. Before Opening Any Web UI

Folder layout, permissions and networking. Nothing below works properly if this step is skipped.

---

## 0.1 Folder structure

This is the single most important decision in the whole setup. Get it wrong and hardlinks
stop working, every import becomes a copy, disk usage doubles and I/O saturates.

The Servarr wiki is explicit about this: passing two separate volumes such as `/tv`, `/movies`
and `/downloads` makes them look like different filesystems even when they are one filesystem
outside the container. Hardlinks then fail, and instead of an instant atomic move you get a
slow, I/O-heavy copy+delete.

The layout expected by this guide:

```
/home/ubuntu/media
├── torrents                   # downloads (qBittorrent)
│   ├── movies
│   ├── tv
│   └── incomplete
└── library                    # final library (Radarr/Sonarr write, Jellyfin reads)
    ├── movies
    └── tv
```

**How `compose-medias.yml` maps it — this part is already correct:**

| Service | Host path | Container path |
|---|---|---|
| radarr | `/home/ubuntu/media` | `/media` |
| sonarr | `/home/ubuntu/media` | `/media` |
| qbittorrent | `/home/ubuntu/media/torrents` | `/media/torrents` |
| jellyfin | `/home/ubuntu/media/library` | `/media/library` |
| bazarr | `/home/ubuntu/media/library` | `/media/library` |
| prowlarr | — | — (never touches files) |
| flaresolverr | — | — (never touches files) |

Two properties make this work:

1. **Radarr and Sonarr are the ones that create hardlinks**, and they mount the whole tree as a
   single `/media` volume — so `/media/torrents` and `/media/library` are the same filesystem
   from their point of view. That is the condition for hardlinking.
2. **Every container agrees on the path string.** What qBittorrent calls
   `/media/torrents/movies/x.mkv` is the exact same path inside Radarr. No path mapping is ever
   needed anywhere in the stack.

The rule from the wiki: your download folder and your media (library / root) folder cannot be
the same location — but they must live on the **same volume**, which is exactly the case here.

## 0.2 Create the folders

```bash
mkdir -p /home/ubuntu/media/{torrents/{movies,tv,incomplete},library/{movies,tv}}
sudo chown -R 1000:1000 /home/ubuntu/media
chmod -R 775 /home/ubuntu/media
```

Every container runs with `PUID=1000` / `PGID=1000`, so permissions stay uniform. No need to
touch `UMASK` or the "Set Permissions" options inside Radarr/Sonarr.

## 0.3 Container-to-container communication

Containers in the same Compose file resolve each other by **service name**. Always use that —
never `localhost` or `127.0.0.1`. The Bazarr wiki warns about this specifically: a loopback
address points at the container itself, not at the Docker host.

| Service | Internal hostname | Port |
|---|---|---|
| qBittorrent | `qbittorrent` | 8080 |
| Prowlarr | `prowlarr` | 9696 |
| Radarr | `radarr` | 7878 |
| Sonarr | `sonarr` | 8989 |
| Bazarr | `bazarr` | 6767 |
| Jellyfin | `jellyfin` | 8096 |
| FlareSolverr *([2.3](02-prowlarr-indexers.md#23-flaresolverr-optional))* | `flaresolverr` | 8191 |

Note that `compose-utils.yml` is a **separate stack with its own network**, so Uptime Kuma must
monitor these services by `http://SERVER_IP:<port>`, not by service name.

## 0.4 Configuration order

1. [qBittorrent](01-qbittorrent.md) — paths, categories, exclusions
2. [Prowlarr — part 1](02-prowlarr-indexers.md) — authentication + indexers (+ FlareSolverr, only if an indexer needs it)
3. [Radarr](03-radarr.md) — needs qBittorrent ready
4. [Sonarr](04-sonarr.md) — same
5. [Prowlarr — part 2](05-prowlarr-apps.md) — Apps; needs the Radarr/Sonarr API keys
6. [Bazarr](06-bazarr.md) — needs Radarr/Sonarr
7. [Jellyfin](07-jellyfin.md) — libraries + API keys
8. [Final integrations](08-integrations.md) — notifications → Jellyfin

---

Next: [1. qBittorrent](01-qbittorrent.md)
