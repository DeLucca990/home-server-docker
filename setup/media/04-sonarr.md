# 4. Sonarr

`http://SERVER_IP:8989`

Prerequisite: [1. qBittorrent](01-qbittorrent.md). Same logic as [Radarr](03-radarr.md)
throughout — `Forms` authentication, API key at `Settings → General → Security`.

---

## 4.1 Media Management

`Settings → Media Management` (with "Show Advanced" on).

### Episode Naming

`Rename Episodes` ✅

Standard Episode Format (TRaSH):

```
{Series CleanTitleWithoutYear} {(Series Year)} - S{season:00}E{episode:00} - {Episode CleanTitle:90} {[Custom Formats]}{[Quality Full]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo VideoCodec]}{-Release Group}
```

Daily Episode Format:

```
{Series CleanTitleWithoutYear} {(Series Year)} - {Air-Date} - {Episode CleanTitle:90} {[Custom Formats]}{[Quality Full]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo VideoCodec]}{-Release Group}
```

Series Folder Format (Jellyfin uses TVDb for series — it does not support IMDb IDs for shows):

```
{Series CleanTitleWithoutYear} {(Series Year)} [tvdbid-{TvdbId}]
```

Season Folder Format: `Season {season:00}`
Multi Episode Style: `Prefixed Range`

> A wiki warning worth repeating: if you leave quality/resolution or release group out of the
> naming scheme, that information cannot be recovered later.

### Importing

| Field | Value |
|---|---|
| Episode Title Required | `Never` (imports TBA episodes immediately) |
| Use Hardlinks instead of Copy | ✅ |
| Import Extra Files | ✅ — `srt,ass,sub,idx,nfo` |

### File Management

`Analyze video files` ✅

### Root Folders

`/media/library/tv`

## 4.2 Profiles and Download Client

Profile: clone `HD-1080p` with the same criteria as Radarr, including the same negative Custom
Format for x265 (`x265 (HD)` at **-10000**) — same hardware reason.

Download Client: qBittorrent, host `qbittorrent`, port `8080`, **Category `sonarr`**,
`Remove Completed` ✅.

Indexers: empty — they come from [Prowlarr](05-prowlarr-apps.md).

---

Previous: [3. Radarr](03-radarr.md) · Next: [5. Prowlarr — Apps](05-prowlarr-apps.md)
