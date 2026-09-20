# 8. Final Integrations — Notify Jellyfin

Without this, new media only shows up in Jellyfin on the next scheduled scan.

Prerequisite: the API keys created in [7.4](07-jellyfin.md#74-api-keys).

---

## Radarr

`Settings → Connect → + → Emby / Jellyfin`

| Field | Value |
|---|---|
| Name | Jellyfin |
| On Import / On Upgrade / On Rename / On Movie Delete | ✅ |
| Host | `jellyfin` |
| Port | `8096` |
| Use SSL | ❌ |
| API Key | the "Radarr" key created in Jellyfin |
| Update Library | ✅ |

**Test** → **Save**.

## Sonarr

`Settings → Connect → + → Emby / Jellyfin` — identical, using the "Sonarr" key and the triggers
`On Import`, `On Upgrade`, `On Rename`, `On Series Delete`, `On Episode File Delete`.

## Bazarr

Already done in [step 6.6](06-bazarr.md#66-jellyfin-native-integration).

---

Previous: [7. Jellyfin](07-jellyfin.md) · Next: [9. Common problems](09-troubleshooting.md)
