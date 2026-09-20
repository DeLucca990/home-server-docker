# 7. Jellyfin

`http://SERVER_IP:8096`

> ⚠️ **Mind the version.** Jellyfin jumped from 10.11 straight to **12.0** (Sep 2026) — the leading
> "10." was dropped from the versioning scheme. Since `compose-medias.yml` uses `:latest`, a
> `docker compose pull` can trigger that migration. The release notes require a **full backup of
> the data directory first** (`config/jellyfin/`), because database changes make rollback
> impossible without a restore, and a **full library scan afterwards**. Repository plugins should
> be removed before and re-added after. 12.0 also ships FFmpeg 8.1 and the "Modern" UI by default
> — re-check the hardware acceleration page after upgrading.

---

## 7.1 Setup wizard

Language, admin user, password. For libraries, add:

| Content type | Name | Folder |
|---|---|---|
| Movies | Movies | `/media/library/movies` |
| Shows | TV | `/media/library/tv` |

**Never point a library at `/media/library` as a whole** — keep movies and TV as separate
libraries so Jellyfin picks the right metadata provider for each. (The Jellyfin container only
mounts `/media/library`, so downloads in progress are out of reach by construction.)

For each library:

- **Preferred metadata language**: `Português (Brasil)` · **Country**: `Brazil`
- **Enable real time monitoring**: ❌ — Radarr/Sonarr will notify
  ([step 8](08-integrations.md)), which is more reliable than inotify over a bind mount
- **Extract chapter images**: ❌ — heavy FFmpeg work on the CPU with little payoff

## 7.2 Subtitles

`Dashboard → Playback` and in the user settings:

- Preferred subtitle language: `por` / Portuguese (Brazil)
- Subtitle mode: `Always show` or `Only forced`
- Avoid any option that forces burn-in of text subtitles — that kills direct play and pushes
  everything into transcoding

## 7.3 Transcoding

The container already passes `/dev/dri` through and sets `GIDLIST=109` for the `render` group.
If VAAPI is already configured against `/dev/dri/renderD128`, leave it alone. Just confirm after
any update that **HEVC is not ticked** in the hardware decode list — the HD 3000 has no such
block, and enabling it makes FFmpeg try and then fall back to software.

## 7.4 API Keys

`Dashboard → API Keys → +`

Create three separate keys (easier to revoke individually): `Radarr`, `Sonarr`, `Bazarr`.

The first two are used in [step 8](08-integrations.md); the third goes back into
[Bazarr 6.6](06-bazarr.md#66-jellyfin-native-integration).

---

Previous: [6. Bazarr](06-bazarr.md) · Next: [8. Final integrations](08-integrations.md)
