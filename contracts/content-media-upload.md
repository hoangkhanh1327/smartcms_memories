---
title: 'Content media upload (signed, direct to CDN)'
tags:
  - api
  - web
  - content
  - contract
  - upload
summary: >-
  Per-slot image upload for VOD/Movie/Music Video forms; server validates,
  resizes variants, pushes to CDN, returns paths + HMAC token used in
  MEDIA_TOKENS on save.
status: ready
links:
  - contracts/content-edit.md
  - contracts/content-cate.md
updated: '2026-10-16'
---
# Content media upload — feature 008

Code: `api: src/modules/content/shared/media-upload/` (`MediaUploadModule`, imported directly by
`app.module` since feature 009); web client `src/components/shared/contentEdit/mediaUpload.ts` +
`ImageSlotField`.

## Endpoints

`POST /content/media-upload/{vod|movie|music-video}/{create|update}` — multipart `file`, `slot`
(neutral image field, e.g. `HOR_POSTER_APP`, `BIGBANNER`, `POSTER_SHUFFLE_VER`), optional
`TYPE_ID` (Movie type 21 shuffle sizes). Each route reuses the kind's existing permission
(`vod-index.create/update`, `movies-index.*`, `music-clip-index.*`) under that kind's menu.

Response (`ContentEditResponseInterceptor` envelope) `data`: `{ fields: { <neutral field>: <CDN
path>, ... }, token }`.

## Behaviour

- Slot rules (size / max bytes / png) come from the old VOD, Movie, Music controllers
  (`media-upload-rules.ts`). Errors → 400 Vietnamese; CDN failure → 502.
- Server-made variants are returned in `fields`: `HOR_POSTER_APP` → `HOR_POSTER` (890×500) for
  VOD/Movie; `BIGBANNER` → `VER_POSTER` (400×600), plus `VER_POSTER_APP` for VOD.
- Files go straight to the CDN (no temp area, no DB rows — no DB permission, multi-instance API);
  images from cancelled edits stay on the CDN (accepted).
- `token` = stateless HMAC (key derived from `JWT_SECRET` with the fixed label
  `content-media-upload`, 24h TTL) over `{kind, slot, fields, userId}`. On save the client sends
  tokens in `MEDIA_TOKENS`; the server rejects new images without a valid token of the same kind
  and user. A used token **fills only the fields the client did not send** (e.g. a variant);
  any value the client sent — including `null` for a removed slot — wins (fixed 2026-10-07: a
  freshly uploaded image removed before saving used to come back).
- Every upload writes an audit log entry (`module: CONTENT_MEDIA_UPLOAD`).
- Local env (`env=local`) skips the CDN and returns the API-served path.

## Category kinds (feature 011, 2026-10-16)

`MediaKind` = content kinds + `VOD_CATE` | `MOVIE_CATE` | `MUSIC_CATE`. Routes `POST content/media-upload/{vod-cate|movie-cate|music-cate}/{create|update}`
reuse `vod-category.*`, `movies-category.*`, `music-clip-cate.*` create/update keys. Slots: VOD `CATE_POSTER_B2B`, `CATE_DEFAULT_POSTER`;
Movie `CATE_POSTER_B2B`; Music `CATE_POSTER` — no size/format limit beyond `imageFileFilter` (as before). Tokens are bound to the
kind, so a content token can't be used on a category and vice versa. See `contracts/content-cate.md`.
