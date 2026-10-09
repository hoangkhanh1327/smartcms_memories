---
title: content expire date
tags:
  - contracts
summary: '# Content bulk expire date — API contract'
status: ready
links:
  - repos/api.md
  - architecture/decisions/0004-permission-model.md
updated: '2026-10-09'
---
# Content bulk expire date — API contract

Shared service `ContentExpireDateService` (api: `src/modules/content/shared/content-expire-date/`), FE shared form `src/components/shared/expireDate/`.

## Endpoints (all return the standard envelope, `data` = result below)

| Route | Permission | Table / scope | Stamps USER_NAME + MODIFYDATE |
|---|---|---|---|
| `POST content/movie/expire-date/preview\|apply` | `movies-index.update` | MOVIE, all TYPE_ID | yes |
| `POST content/vod/expire-date/preview\|apply` | `vod-index.update` | CONTENT, `TYPE_ID NOT IN (75, 88)` (same as VOD list) | yes |
| `POST content/music-video/expire-date/preview\|apply` | `music-clip-index.update` | MUSIC_VIDEO, all TYPE_ID | yes |
| `POST internal/expire-date/preview\|apply` | InternalUserGuard (hardcoded username whitelist) | body `source`: `movie`/`content`/`music_video`, all TYPE_ID | **no** (silent fix) |

Body: `{ items: [{ id, expire_date: 'YYYY-MM-DD[ HH:mm[:ss]]' }] }` (≤ 5000, ids unique — else 400 with Vietnamese message). Internal adds `source`; content endpoints ignore it.

Preview → `{ mode, source, total, to_update, unchanged, already_expired, not_found[], items[{id,name,type_id,status,current,next,unchanged}] }`. IDs outside the screen scope are `not_found`.
Apply → `{ mode, source, updated[], unchanged, not_found[], popular_updated, redis_scheduled, crontab_keywords[] }`.

## Write rules (mirror each table's CMS save flow)
- CONTENT stores date only; MOVIE / MUSIC_VIDEO store datetime. Date-only input → `00:00:00` (kept from the legacy private endpoint, confirmed by user 2026-10-09).
- MOVIE / MUSIC_VIDEO also update `TPW_POPULAR.EXPIRE_DATE` (VOD_DETAIL_TYPE 2 / 3); CONTENT does not.
- Redis detail write/reset per status + crontab per TYPE_ID via `ContentCacheRefreshService` (`content/shared/content-cache`).
- Logs: one row per content, one per changed TPW_POPULAR row, one summary row per run (object_id null).

Replaced the unauthenticated-permission `POST /private/update-music-video-expire-date` (removed 2026-10-09).
