# 6. Bazarr

`http://SERVER_IP:6767`

Prerequisites: [Radarr](03-radarr.md) and [Sonarr](04-sonarr.md) working, with their API keys.
Section 6.6 additionally needs the Jellyfin API key from [step 7](07-jellyfin.md).

---

## 6.1 General

`Settings → General`:

- Set authentication (`Forms`) and credentials
- Note the API key (only needed if you integrate other tools)

## 6.2 Sonarr and Radarr

`Settings → Sonarr` → **Enabled** ✅

| Field | Value |
|---|---|
| Address | `sonarr` |
| Port | `8989` |
| Base URL | *(empty)* |
| API Key | Sonarr's |
| SSL | ❌ |
| Download Only Monitored | ✅ |

Click **Test** — do not proceed until it passes. Save.

Repeat under `Settings → Radarr` with `radarr` / `7878`.

**Minimum Score**: start at 90 (series) and 70 (movies). If subtitles come out frequently out of
sync or plain bad, raise the score.

**Path Mappings: leave EMPTY.** Bazarr mounts the library at `/media/library`, the exact path
Radarr and Sonarr report — so mapping here only breaks things. The wiki is emphatic: if both
sides have the same value you do not need it, and leaving it in will produce an error.

## 6.3 Languages

`Settings → Languages`:

1. **Subtitles Language** → add `Portuguese (Brazil)` and optionally `English`
2. **Languages Profiles** → `Add New Profile`:
   - Name: `PT-BR`
   - Languages: `Portuguese (Brazil)` then `English` (in that order)
   - **Cutoff**: `Portuguese (Brazil)`
   - Leave HI (Hearing Impaired) and Forced unchecked on the first entry
3. **Default Settings** → apply the `PT-BR` profile automatically to Series and Movies

How the cutoff works: with the profile above, if pt-BR is found it downloads it and stops; if not,
it keeps going through English until pt-BR shows up.

> ⚠️ **Default Settings only applies going forward** — the profile is auto-applied to series and
> movies added to Bazarr *after* you enable the option. For anything already there, use bulk edit
> on the Series/Movies tabs.

## 6.4 Providers

`Settings → Providers → +`

Enable several. Options that work well for pt-BR today:

- **OpenSubtitles.com** — free account required (not the legacy .org)
- **Subdl** — good pt-BR coverage
- **Podnapisi**
- **Gestdown (Addic7ed)** — TV only

Ignore older guides that recommend Subscene: the service shut down in 2024.

## 6.5 Subtitles

`Settings → Subtitles`:

| Field | Value |
|---|---|
| Subtitle Folder | `AlongSide Media File` |
| Upgrade Previously Downloaded Subtitles | ✅ |
| Use Embedded Subtitles | ✅ (avoids re-downloading what is already inside the MKV) |
| Automatic Subtitles Synchronization | ✅ |
| Encode Subtitles to UTF-8 | ✅ (prevents broken accented characters) |
| Use Multithreading | ❌ (your CPU will thank you) |

## 6.6 Jellyfin (native integration)

Recent Bazarr versions integrate directly with Jellyfin to refresh items and libraries after
subtitle changes — downloads, uploads, deletions — authenticating with an API key. Requires
Bazarr 1.5.7-beta.26 or newer.

Configure this **after** creating the Jellyfin API key ([step 7.4](07-jellyfin.md#74-api-keys)):

`Settings → Jellyfin`:

- **Enabled** ✅
- **Server URL**: `http://jellyfin:8096`
- **API Key**: the one created in Jellyfin
- **Test** → on success the button shows the server name and version
- **Movie Library** / **Series Library**: select yours
- **Refresh movie/series metadata after downloading subtitles** ✅
- **Refresh method**: `Immediate` — re-reads item metadata right away without contacting external
  providers; recommended for most setups

Only `movies` and `tvshows` collection types appear in the dropdown; mixed-content libraries are
filtered out by design.

---

Previous: [5. Prowlarr — Apps](05-prowlarr-apps.md) · Next: [7. Jellyfin](07-jellyfin.md)
