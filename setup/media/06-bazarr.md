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
sync or plain bad, raise the score (and see [6.5.1](#651-audio-synchronization-ffsubsync) —
out-of-sync subtitles are better fixed by automatic synchronization than by chasing scores).

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
| Automatic Subtitles Synchronization | ✅ (details in [6.5.1](#651-audio-synchronization-ffsubsync)) |
| Encode Subtitles to UTF-8 | ✅ (prevents broken accented characters) |
| Use Multithreading | ❌ (your CPU will thank you) |
| Skip video file hash calculation | ❌ (keep it off: the hash is what lets providers match your exact release) |

### 6.5.1 Audio synchronization (ffsubsync)

**The problem:** a subtitle is often timed for a different release of the episode (another encode,
cut or source), so it does not match the file's hash and drifts out of sync. Bazarr can fix this
automatically with **ffsubsync**, which compares the video's audio against the subtitle and shifts
the timing — it works even when the hash does not match.

`Settings → Subtitles → Audio Synchronization`:

| Field | Value |
|---|---|
| Enable Automatic Subtitles Audio Synchronization | ✅ |
| Series Score Threshold For Audio Sync | `97` (see note below) |
| Movies Score Threshold For Audio Sync | `70` (raise only if movies also drift) |
| Providers to Exclude from Automatic Synchronization | *(empty)* |

> **How the threshold works:** only subtitles with a score **below** the value are synchronized.
> A subtitle from a different release but with a similar name, season, group and source can score
> above 90 without matching the hash, so it would be accepted *without* sync. To tune it, open
> `History`, check the scores of the subtitles that came out of sync, and set the threshold above
> the highest of them.

`Settings → Subtitles → Advanced FFsubsync Options`:

| Field | Value |
|---|---|
| Synchronization Reference | `Use Audio Track as Reference` |
| Prefer Original Language Audio Track | ✅ if your files carry both original and dubbed audio (subtitles are usually timed to the original; falls back to the default track if there is no original) |
| Do Not Fix Framerate Mismatch | ❌ **keep it OFF.** With it on, ffsubsync is forbidden from correcting 23.976 vs 25 fps differences, a very common cause of subtitles that start fine and slowly drift |
| Golden-Section Search | ❌ (slower and heavier on the CPU; use it only for manual syncs when the automatic one fails) |
| Max Offset Seconds | `60` |
| Generate Debug File Instead of Synchronizing | ❌ |

> ⚠️ **Only subtitles downloaded after enabling this are synced automatically.** Subtitles already
> on disk need a manual sync (below).

#### Manual sync (existing subtitles)

Open the series/movie, then the episode, and use the subtitle's menu → **Sync**. In the dialog:

| Field | Value |
|---|---|
| Reference | `a:0` (first audio track). Change it only if the original-language audio is on another track |
| Max Offset Seconds | empty on the first try. If the result is wrong, retry with ~`60` (or `15`–`30` for small constant offsets) |
| No Fix Framerate | ☐ unchecked |
| Golden-Section Search | ☐ unchecked at first. If the result is still off, retry with it checked |

To fix many episodes at once, select several on the series page and use the bulk subtitle tools,
which include **Sync** (exact menu name varies by version).

> ⚠️ **Low-power servers:** ffsubsync analyzes the audio and is CPU-heavy. Run bulk syncs in small
> batches (one series at a time), away from transcoding. If sync keeps failing, check
> `System → Logs` for ffsubsync errors or low-confidence results; sometimes the subtitle is simply
> from a very different version of the episode.

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

> Jellyfin does not fix subtitle timing by itself. Let Bazarr deliver the already-synchronized file
> to disk and use Jellyfin's player offset only as a last-resort manual workaround.

---

Previous: [5. Prowlarr — Apps](05-prowlarr-apps.md) · Next: [7. Jellyfin](07-jellyfin.md)