# 3. Radarr

`http://SERVER_IP:7878`

Prerequisite: [1. qBittorrent](01-qbittorrent.md) configured and running.

---

## 3.1 Authentication

Same screen as Prowlarr: `Forms (Login Page)` plus credentials. Then copy the API key from
`Settings → General → Security → API Key` — you will need it in Prowlarr and Bazarr.

## 3.2 Media Management

`Settings → Media Management` — **turn on "Show Advanced"** at the top, or half the options stay
hidden.

### Movie Naming

| Field | Value |
|---|---|
| Rename Movies | ✅ |
| Replace Illegal Characters | ✅ |

Standard Movie Format (TRaSH scheme for Jellyfin, using TMDb):

```
{Movie CleanTitle} {(Release Year)} [tmdbid-{TmdbId}] - {{Edition Tags}} {[MediaInfo 3D]}{[Custom Formats]}{[Quality Full]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo VideoCodec]}{-Release Group}
```

Movie Folder Format:

```
{Movie CleanTitle} ({Release Year}) [tmdbid-{TmdbId}]
```

The bracketed ID makes Jellyfin match the item by ID instead of by text search. One caveat from
TRaSH worth noting: folder names are written to the database when the movie is added to Radarr,
and the ID may be missing or wrong at that moment, which can leave you with a blank ID in the
folder name. Putting the ID in the *filename* means it gets re-fetched on every download or
upgrade.

### Importing

| Field | Value |
|---|---|
| Use Hardlinks instead of Copy | ✅ |
| Import Extra Files | ✅ — extensions: `srt,ass,sub,idx,nfo` |

### File Management

| Field | Value |
|---|---|
| Analyze video files | ✅ (required for the MediaInfo tokens in the naming scheme) |
| Unmonitor Deleted Movies | ✅ |
| Set Permissions | ❌ (unnecessary — everything runs as UID 1000) |

### Root Folders

`+ Add Root Folder` → `/media/library/movies`

## 3.3 Profiles

`Settings → Profiles`. The wiki's recommendation is to build your own profiles and select only
the quality sources and languages you actually want.

Practical starting point: clone `HD-1080p` and under *Qualities* keep only `Bluray-1080p`,
`WEBDL-1080p`, `WEBRip-1080p`, `HDTV-1080p`, with *Upgrade Until* set to `Bluray-1080p`.

> **Important on this hardware.** The Intel HD 3000 has no HEVC decode block — any x265 file falls
> back to software transcoding and saturates the CPU. To stop Radarr grabbing x265 by default,
> import the TRaSH Custom Format **`x265 (HD)`** and give it a score of **-10000** in this profile.
> 1080p HEVC releases are then rejected automatically and Jellyfin stays on the VAAPI/H.264 path.

Also worth importing are the defensive Custom Formats the wiki itself recommends: `BR-DISK`
(avoids poorly named BR-DISKs) and `DV (WEB-DL)` (avoids Dolby Vision without fallback, which
plays with a green tint). Useful tool for keeping this version-controlled:
[Recyclarr](https://recyclarr.dev/).

## 3.4 Download Clients

`Settings → Download Clients → + → qBittorrent`:

| Field | Value |
|---|---|
| Name | qBittorrent |
| Host | `qbittorrent` |
| Port | `8080` |
| Use SSL | ❌ |
| Username / Password | the ones you set |
| Category | `radarr` |

**Test** → should go green. Save.

On the same page, under **Completed Download Handling**: `Remove Completed` ✅.

## 3.5 Indexers

Leave this **empty**. Prowlarr populates it in [step 5](05-prowlarr-apps.md).

---

Previous: [2. Prowlarr — indexers](02-prowlarr-indexers.md) · Next: [4. Sonarr](04-sonarr.md)
