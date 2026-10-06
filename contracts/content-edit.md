---
title: Content edit contract (VOD / Movie / Music Video)
tags:
  - api
  - web
  - content
  - contract
summary: >-
  Neutral JSON contract for create/detail/update of the three content kinds;
  PATCH sends only changed fields; images via signed uploads.
status: draft
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
updated: '2026-10-06'
---
# Content edit contract — feature 008 (draft)

Source of truth in code: `api: src/modules/content/content-edit-shared/content-edit-fields.ts`
(field catalogue). Mirrored by `web: src/components/shared/contentEdit/types.ts` (phase 1 web).

## Kinds and endpoints (paths unchanged, permissions unchanged)

| | VOD | Movie | Music Video |
|---|---|---|---|
| detail | `GET /content/:id` | `GET /content/movie/:id` | `GET /content/music-video/:id` |
| create | `POST /content` | `POST /content/movie` | `POST /content/music-video` |
| update | `PATCH /content/:id` | `PATCH /content/movie/:id` | `PATCH /content/music-video/:id` |
| permission names | `vod-index.detail/create/update` | `movies-index.*` | `music-clip-index.*` |

During rollout VOD and Movie keep `PUT` on the update path returning 409 "Trang đã có phiên bản
mới, vui lòng tải lại trang" (stale bundles); removed after 1–2 releases.

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

## Images

See `contracts/content-media-upload.md`. Every new image path in a body must be covered by a token
in the transient field `MEDIA_TOKENS`; unchanged / `null` images need none.

## Update semantics

Server loads the current row, applies the patch, and computes every branch / derived value from
the merged row; only patched columns are written; link tables (categories, actors) are rewritten
only when their field is present. Audit log (before/after per patched column) is written before
the response.
