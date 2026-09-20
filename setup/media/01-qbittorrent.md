# 1. qBittorrent

`http://SERVER_IP:8080`

Prerequisite: [0. Before opening any Web UI](00-before-you-start.md)

---

## 1.1 First login

The LinuxServer image **generates a random temporary password at boot** and prints it to the log:

```bash
docker logs qbittorrent | grep -i "password"
```

Username: `admin`. If you do not change the password through the Web UI, a new one is generated
on every container start — so change it right away.

`Tools → Options → Web UI`:

- **Authentication → Username / Password**: set your own credentials
- **Bypass authentication for clients on localhost**: leave unchecked

## 1.2 Downloads

`Tools → Options → Downloads`:

| Field | Value |
|---|---|
| Default Save Path | `/media/torrents` |
| Keep incomplete torrents in | `/media/torrents/incomplete` (checked) |
| Copy .torrent files for finished downloads | unchecked |
| Torrent content layout | `Original` |
| Run external program on completion | unchecked |

## 1.3 Excluded file names

Same page, under **Excluded file names**. Tick the checkbox and add one pattern per line:

```
*.exe
*.scr
*.bat
*.cmd
*.msi
*.lnk
*.com
*.vbs
*.pif
```

Matching files are flagged as "do not download" and are skipped entirely. This is a meaningful
safety net: fake releases on public trackers routinely ship an executable disguised as the
media file — `.scr` and `.lnk` are the classic ones, since they look harmless and still execute.
As of qBittorrent 5.x the patterns apply to folder names too.

Two caveats worth knowing:

- **It is not a virus scanner.** It filters by filename, nothing more. A malicious `.mkv`
  exploiting a player vulnerability passes straight through.
- **Verify it actually applies to torrents added by Radarr/Sonarr.** There is a history of bugs
  in the 5.x line where the exclusion list was ignored for torrents added via magnet links,
  RSS rules, or the WebUI/API — which is exactly how the `*arr` apps add them. After the first
  automated grab, open the torrent's **Content** tab and confirm any matching files show as
  "Do not download". If they do not, treat the setting as best-effort and rely on indexer
  quality plus the Radarr/Sonarr Custom Formats instead.

A useful companion setting for junk: Radarr/Sonarr will refuse to import a release whose only
video file is missing or undersized, so an excluded executable usually turns into a failed
import rather than a silent mess.

## 1.4 Categories

Create two categories (right-click "Categories" in the sidebar → *Add category*):

| Category | Save path |
|---|---|
| `radarr` | `/media/torrents/movies` |
| `sonarr` | `/media/torrents/tv` |

These are the exact names you will enter in Radarr/Sonarr later.

## 1.5 Seeding limits

`Tools → Options → BitTorrent → Seeding Limits`:

- Tick "When ratio reaches" and/or "When seeding time reaches" with your targets
- **Action: `Pause torrent`** — never "Remove torrent"

This matters: with *Remove Completed* enabled, Radarr/Sonarr will only delete the torrent if the
client reports that seeding has finished and the torrent is stopped. If qBittorrent removes it
on its own, the `*arr` app loses track of it.

## 1.6 Incoming connections port

`compose-medias.yml` already publishes `6881` (TCP and UDP) and sets `TORRENTING_PORT=6881`.
Confirm the Web UI agrees: `Options → Connection → Port used for incoming connections` = `6881`.

Without an open listening port you only make outbound connections, which hurts speed
considerably.

---

Previous: [0. Before you start](00-before-you-start.md) · Next: [2. Prowlarr — indexers](02-prowlarr-indexers.md)
