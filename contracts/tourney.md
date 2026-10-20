---
title: Tournaments (TOURNEY) and tournament categories (TOURNEY_CATE)
summary: >-
  Feature 013: "Danh sách giải đấu" is TOURNEY (path/permissions
  `tournament-cate.*` kept), each tournament owns one CONTENT_CATE_SUB root
  (SUB_TYPE_ID 1, via TOURNEY_ID); "Thể loại giải đấu" is TOURNEY_CATE (TYPE_ID
  89) as the 4th content-cate kind; categories per tournament in
  TOURNEY_CATE_LIST.
tags:
  - contract
  - api
  - web
  - content
  - tournament
status: ready
links:
  - contracts/content-cate-sub.md
  - contracts/content-cate.md
updated: '2026-10-20'
---
# Tournaments — feature 013

## Storage
- `TOURNEY`: TOURNEY_ID, TOURNEY_NAME, LOGO_TV / LOGO_TV_SMALL / LOGO_PHONE, SORT_ORDER, START_DATE, END_DATE,
  STATUS (0 chưa diễn ra / 1 đang diễn ra / 2 đã kết thúc), URL_GET_DATA (never written by CMS), TOURNAMENT_ID (SportHub),
  SPORT_ID (SPORTS), CREATED_AT, MODIFIED_AT, USER_*.
- `TOURNEY_CATE_LIST`: (CONTENT_ID = TOURNEY_ID, CATE_ID) composite PK — a tournament has many categories.
- `TOURNEY_CATE`: 011 column convention (CATE_ORDER, CATE_STATUS, TYPE_ID fixed **89**, CREATEDATE/MODIFYDATE,
  HIDDEN_DEVICE_LIST + V2, CATE_CODE, PARENT_ID). VOD Common TPW group 14 reads it.

## API — `content/tournament-cate` (module `src/modules/content/tourney/`, menu `tournament-cate`, names kept)
- `GET /` (?TOURNEY_NAME, ?STATUS 0|1|2, ?VISIBLE 0|1) rows + `CATE_ID` (root), `VISIBLE` (root STATUS, null when no
  root), `TOURNEY_CATE_IDS`, `TOURNEY_CATES` — three parallel reads, one DB round trip (p99 82 ms on stage).
- `GET :id` · `POST /` JSON `{ TOURNEY_NAME*, LOGO_PHONE*, LOGO_TV*, LOGO_TV_SMALL*, START_DATE* (YYYY-MM-DD), END_DATE?,
  STATUS? (default 0), SORT_ORDER? (default max+1), SPORT_ID?, TOURNAMENT_ID?, TOURNEY_CATE_IDS? number[], VISIBLE?
  (default 1), MEDIA_TOKENS }` — one tx: TOURNEY → root CONTENT_CATE_SUB → TOURNEY_CATE_LIST.
- `PATCH :id` changed fields only; name / sport sync to the root; `VISIBLE` cascades the root subtree; categories
  diff-synced; missing root recreated. END_DATE ≥ START_DATE (400). Name unique case-insensitive (422).
- `PATCH :id/status { STATUS: 0|1|2 }` · `PATCH :id/visibility { STATUS: 0|1 }` (permission `tournament-cate.update-status`)
  · `DELETE :id` → 422 while the root has children or content links, else tx: categories → root → TOURNEY.
- Old `PUT :id` / `PUT update-status/:id` → 409 stubs; old `vods/:id` routes removed (linked content is on the
  sub-cate screen). Logos: `POST content/media-upload/tourney/{create,update}` (kind TOURNEY: 1080×1080, 1920×1080, 80×80).
- Audit module `TOURNEY`; writes queue the `tournament` crontab (stage has no API definition for it).

## API — `content/tourney-cate` (menu `tourney-cate`) and lookups
- CATE_KINDS `TOURNEY` in the shared content-cate service: list / detail / POST / PATCH / `PATCH :id/status
  { CATE_STATUS }` / `PUT sort-order`; no delete; no EN name / images.
- `GET lookup/sports` (`{ value, label }`), `GET lookup/tourney-cates` (`{ value, label, parentId, nodeType }`) — JWT only.

## Web
- `VOD/Tournament` (Danh sách giải đấu, `/content/tournament`), `VOD/TourneyCate` (`/content/tourney-cate`) under the
  Sporthub menu; tournament-homepage routes removed. MatchSchedule's LiveScore list moved to its own `LivescoreProvider`.
