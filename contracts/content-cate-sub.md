---
title: Secondary categories ("Danh mục phụ") — Anime first
summary: >-
  Anime genres live in CONTENT_CATE_SUB (SUB_TYPE_ID = 2) with links in
  CONTENT_CATE_SUB_LIST; anime-cate API restandardised like 011 (JSON, PATCH
  diff, explicit status). Tournament stays on TOURNAMENT_CATE for now.
tags:
  - contract
  - api
  - web
  - content
  - anime
  - cate-sub
status: ready
links:
  - contracts/content-cate.md
  - contracts/content-edit.md
updated: '2026-10-19'
---
# Secondary categories ("Danh mục phụ") — feature 012

## Storage
- `CONTENT_CATE_SUB`: tree per `SUB_TYPE_ID` (`CONTENT_TYPE_SUB`: 1 Thể thao, 2 Anime). Columns: CATE_ID, CATE_NAME, CATE_CODE
  (`'0'` root, `'0_<root>_…_<parent>'`), PARENT_ID, CATE_NODE_TYPE (1 root / 2 child, server), SORT_ORDER, STATUS (0/1),
  HIDDEN_DEVICE_LIST (v1, server-derived) + HIDDEN_DEVICE_LIST_V2, USER_*, CREATED_AT, MODIFIED_AT, GROUP_CATEGORY_ID
  (default 1, never written), SPORT_ID / CATE_LINK (never written). No logo / date / league columns.
- `CONTENT_CATE_SUB_LIST`: ID, CATE_ID, CONTENT_ID, TYPE_ID. CONTENT_ID mixes CONTENT_ID (VOD) and MOVIE_ID (movie);
  always filter by the TYPE_ID group (`resolveCateLinkTypeIds`).
- Anime no longer reads/writes TOURNAMENT_CATE / TOURNAMENT_CATE_LIST (no dual-write). Old-data migration is done
  outside the code, keeps CATE_ID, and ships with the release. Tournament (SUB_TYPE_ID 1) is still on TOURNAMENT_CATE.

## API — `content/anime-cate` (menu_key `anime-cate`, permission names unchanged)
- Shared module `src/modules/content/shared/content-cate-sub/` (`ContentCateSubService`, `ContentCateSubLinksService`,
  registry `SUB_CATE_TYPE_CONFIG`); every query is scoped by SUB_TYPE_ID. Envelope: `ContentEditResponseInterceptor`.
- `GET /` (?CATE_NAME, ?STATUS) full rows, v1→v2 fallback batched · `GET :id` · `POST /` JSON
  `{ CATE_NAME*, PARENT_ID?, SORT_ORDER? (1–10000, default max+1), STATUS? (default 0), HIDDEN_DEVICE_LIST_V2? number[] }`
  · `PATCH :id` changed fields only (empty → 400; `[]` clears devices; numbers cannot be null) · `PATCH :id/status`
  `{ STATUS: 0|1 }` applied to the whole subtree (empty body → 409 reload) · `PUT sort-order { items:[{CATE_ID, SORT_ORDER}] }`
  · `DELETE :id` subtree + its links (one tx) · `GET vods/:id` linked VOD + movie (paged) · `DELETE vods/:id` unlink.
- Old `PUT :id` and `PUT update-status/:id` are 409 stubs ("Trang đã có phiên bản mới, vui lòng tải lại trang").
- Rules: root names unique (case-insensitive) → 422; parent must exist in the same SUB_TYPE_ID; self/descendant parent
  → 422; CATE_CODE > 50 chars → 400; parent move rewrites the subtree prefix in one UPDATE; audit module
  `ANIME_CATE` / `ANIME_CATE_LIST`, awaited.

## Consumers
- Edit forms (008 contract field `TOURNAMENT_CATE_LIST`, name unchanged): `content-tournament-cate.helper` dispatches by
  tree type — Anime TYPE_IDs (2, 6, 20, 22, 25) → CONTENT_CATE_SUB_LIST, tournament (12, 78) → TOURNAMENT_CATE_LIST.
  VOD duplication copies both link tables.
- `lookup/cate-tree-options`: Anime tree from CONTENT_CATE_SUB (STATUS 1); response `{ TYPE_CONTENT, items }` unchanged.
- TPW menu "Cate phụ": right column `typeContent=2` reads CONTENT_CATE_SUB; left COUNT resolves parents per group
  (CATE_IDs of the two tables can collide).
- Excel import auto-links only tournaments (unchanged).

## Code map (after review, 2026-10-19)
- `shared/content-cate-sub/`: `entities/`, `repositories/` (all SQL, every query scoped by SUB_TYPE_ID), `services/`
  (`ContentCateSubService` tree rules — CATE_CODE recomputed in TS, written with one `UPDATE … CASE`;
  `ContentCateSubLinksService`). Writes take a `SubCateActor { user, url }` for the awaited audit.
- Tab-3 links of VOD / movie forms: injectable `ContentCateLinkService` (`shared/tournament-cate/content-cate-link.module.ts`,
  imported by VOD core + Movie modules) — `assertMatchesType`, `syncForContent` (returns tournament-changed flag),
  `csvForContent`, `copySubCateLinks`. The old `content-tournament-cate.helper` is gone.
- `DELETE content/anime-cate/vods/:id` takes a link ID (bad ID → 400 "Mã liên kết nội dung không hợp lệ").
