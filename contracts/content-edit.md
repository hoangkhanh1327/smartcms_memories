---
title: Content edit contract (VOD / Movie / Music Video)
tags:
  - api
  - web
  - content
  - contract
summary: >-
  Neutral JSON contract for create/detail/update of the three content kinds;
  PATCH sends only changed fields; images via signed uploads; standard
  trailer/series sub-table routes.
status: ready
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
  - contracts/content-media-upload.md
updated: '2026-10-06'
---
# Content edit contract — feature 008

Source of truth in code: `api: src/modules/content/content-edit-shared/content-edit-fields.ts`
(field catalogue) and `{vod,movie,music-video}/edit/`. Mirrored by
`web: src/components/shared/contentEdit/` (`types.ts`, `formValues.ts`, `diffValues`).

## Kinds and endpoints (paths unchanged, permissions unchanged)

| | VOD | Movie | Music Video |
|---|---|---|---|
| detail | `GET /content/:id` | `GET /content/movie/:id` | `GET /content/music-video/:id` |
| create | `POST /content` | `POST /content/movie` | `POST /content/music-video` |
| update | `PATCH /content/:id` | `PATCH /content/movie/:id` | `PATCH /content/music-video/:id` |
| permission names | `vod-index.detail/create/update` | `movies-index.*` | `music-clip-index.*` |

`@RouteInfo.path` values stay the legacy strings (e.g. `/:CONTENT_ID`, Movie update `/:movie_id`,
Music `/:musicId`) because permission-sync builds `api_link` from them.

VOD and Movie keep `PUT` on the update path returning 409 "Trang đã có phiên bản mới, vui lòng tải
lại trang" (stale bundles); removed after 1–2 releases (feature 008 T50).

## Response

All edit controllers use `ContentEditResponseInterceptor`: `{ status: 200, result: 0, message:
'Success', data, errors: null }`. Errors are `HttpException`s with Vietnamese messages (400
validation, 404 not found, 409 stale PUT, 502/504 external failure or timeout).

Detail `data` = neutral fields + `RELATION_OPTIONS` (labels for form selects, not contract
fields): `{ TAG_IDS: [{value,label}], ACTOR_IDS?: [{value,label}] }` (no actors for Music).
Movie detail also returns read-only `TOTAL_DURATION` (seconds) and `PROVIDER_NAME`. Tags are
filtered by the content's own `TYPE_ID`.

## Body rules (validated by `ContentEditBodyPipe(kind, mode)`, 400 with Vietnamese messages)

- JSON only. Neutral names (`NAME`, `DESC`, `STATUS`, `VER_POSTER`, …) map to `CONTENT_*` /
  `MOVIE_*` / `VIDEO_*` columns per kind. Unknown, read-only or other-kind fields → 400.
- Absent = unchanged. `null` = clear (strings/images stored as `''`, dates and shuffle posters as
  NULL). Id lists (`CATE_ID_LIST`, `HIDDEN_DEVICE_LIST_V2`, `ACTOR_IDS`, `TOURNAMENT_CATE_LIST`) are
  arrays; clear with `[]`. Required (`NAME`, `NAME_ADMIN`, `SERVICE_ID`, `STATUS`,
  `CATE_ID_LIST`) cannot be cleared and must be present on create. Non-null numeric selects
  (`ADS_BLACKLIST`, `CONTRACT`, …) reject `null` — send the default.
- Dates: `YYYY-MM-DD HH:mm:ss`.
- Read-only in detail: `ID`, `KEYWORD` (server-derived from name), `TAG_IDS` (tags keep their own
  immediate API), `CATEGORY_NAME`, `PENDING_STATUS`, `ACTOR_NAMES`, VOD root fields,
  `CONTENT_MOVIE` (server-derived).
- Shuffle posters (`POSTER_SHUFFLE_VER/HOR`): array of paths per slot (`null` = empty slot). Movie
  stores positionally (3 slots), VOD compacted.
- Intro/outro and "apply to all episodes" are not part of the edit page (plan v3.4/v3.5).

## Images

See `contracts/content-media-upload.md`. Every new image path in a body must be covered by a token
in the transient field `MEDIA_TOKENS`; unchanged / `null` images need none.

## Update semantics

Server loads the current row, applies the patch, and computes every branch / derived value from
the merged row; only patched columns are written; link tables (categories, actors) are rewritten
only when their field is present (batched insert in the transaction). Movie categories are
validated (400) and expanded with their parents. Auto-tags re-run when a trigger field is
patched; Redis write is unconditional. Audit log (before/after per patched column) is written
before the response.

## Sub-tables — standard routes (added next to the legacy ones, which stay)

`{kind}` = `vod` | `movie` | `music-video`. Reads need `<menu>.detail`, writes `<menu>.update`
(`vod-index`, `movies-index`, `music-clip-index`). Legacy sub-table routes had no permission and
are still used by ShortV2 etc.

Trailers — `content/trailers/{kind}`:
- `GET /:parentId` → the single trailer or `null`.
- `POST /` body `{ PARENT_ID, TRAILER_PATH }` or (VOD only) `{ PARENT_ID, COMMON_TRAILER_ID }`.
- `PATCH /:trailerId` body `{ TRAILER_PATH | COMMON_TRAILER_ID }` — VOD only.
- `PATCH /:trailerId/status` body `{ STATUS: 0|1 }` — Movie/Music; VOD returns 400.
- `DELETE /:trailerId`.

Series (episodes) — `content/series/{kind}`:
- `GET /:parentId` → array of episodes.
- `PATCH /status` body `{ PARENT_ID, IDS: number[], STATUS: 0|1 }` (VOD/Movie store inactive as
  -1; the route maps 0 → -1).
- `DELETE /` body `{ PARENT_ID, IDS: number[] }`.
- Episode create/update and a shared series form are a separate feature (plan v3.6).
