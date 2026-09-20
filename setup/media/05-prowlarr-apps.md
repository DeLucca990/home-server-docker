# 5. Prowlarr — Part 2 (Sync the Apps)

Prerequisites: [Radarr](03-radarr.md) and [Sonarr](04-sonarr.md) configured, and both API keys
in hand.

---

## 5.1 Add the apps

Back in Prowlarr: `Settings → Apps → +`

Add **Radarr** and **Sonarr** separately. Only add apps you actually have installed; if you run
multiple instances, each one needs its own entry.

| Field | Radarr | Sonarr |
|---|---|---|
| Sync Level | `Full Sync` | `Full Sync` |
| Prowlarr Server | `http://prowlarr:9696` | `http://prowlarr:9696` |
| Application Server | `http://radarr:7878` | `http://sonarr:8989` |
| API Key | Radarr's | Sonarr's |

On Sync Level: `Full Sync` keeps the app's indexers fully in sync — changes made in Prowlarr
propagate outward, and any change made remotely inside the app gets overwritten by Prowlarr on
the next sync. That is what you want: Prowlarr as the single source of truth.

A detail that trips people up: indexers sync according to the categories they claim to support,
so an indexer that only supports `tv` categories will never reach Radarr. On top of that, the
`*arr` apps only accept indexers whose test queries return results containing at least one of the
configured categories.

The sync runs on save. Check `Radarr → Settings → Indexers` and `Sonarr → Settings → Indexers`:
entries with the `(Prowlarr)` suffix should be there.

## 5.2 Sync Profiles

`Settings → Apps → Sync Profiles` control, per indexer: RSS, interactive search, automatic search
and **Minimum Seeders**. Worth creating a profile with `Minimum Seeders: 3` for public trackers.

---

Previous: [4. Sonarr](04-sonarr.md) · Next: [6. Bazarr](06-bazarr.md)
