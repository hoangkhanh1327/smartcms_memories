---
title: Secondary categories ("Danh mục phụ") — Anime + Giải đấu
summary: >-
  Anime (SUB_TYPE_ID 2, feature 012) and tournaments (SUB_TYPE_ID 1, feature
  013) live in CONTENT_CATE_SUB with links in CONTENT_CATE_SUB_LIST; nothing
  reads/writes TOURNAMENT_CATE* any more. Tournament roots are owned by TOURNEY
  (CONTENT_CATE_SUB.TOURNEY_ID) and locked on the sub-cate screen.
tags:
  - contract
  - api
  - web
  - content
  - anime
  - tournament
  - cate-sub
status: ready
links:
  - contracts/content-cate.md
  - contracts/content-edit.md
updated: '2026-10-20'
---
# Secondary categories ("Danh mục phụ") — features 012 + 013

## Storage
- `CONTENT_CATE_SUB`: tree per `SUB_TYPE_ID` (`CONTENT_TYPE_SUB`: 1 Thể thao / Giải đấu, 2 Anime). Columns: CATE_ID, CATE_NAME,
  CATE_CODE (`'0'` root, `'0_<root>_…_<parent>'`), PARENT_ID, CATE_NODE_TYPE (1 root / 2 child, server), SORT_ORDER,
  STATUS (0/1), HIDDEN_DEVICE_LIST (v1, server-derived) + HIDDEN_DEVICE_LIST_V2, USER_*, CREATED_AT, MODIFIED_AT,
  GROUP_CATEGORY_ID (default 1, never written), SPORT_ID, CATE_LINK, TOURNEY_ID (int, default 0, no index).
- Tournament roots (SUB_TYPE_ID 1, PARENT_ID 0): `TOURNEY_ID` → `TOURNEY.TOURNEY_ID`; CATE_NAME / SPORT_ID copied from
  TOURNEY; STATUS = tournament visibility. Ids are NOT assumed equal for new tournaments (migrated rows happen to match).
- `CONTENT_CATE_SUB_LIST`: ID, CATE_ID, CONTENT_ID, TYPE_ID. CONTENT_ID mixes CONTENT_ID (VOD) and MOVIE_ID (movie);
  always filter by the TYPE_ID group (`resolveCateLinkTypeIds`).
- No code reads/writes TOURNAMENT_CATE / TOURNAMENT_CATE_LIST (no dual-write). Old-data migration is outside the code,
  keeps ids, ships with the release. TOURNAMENT_HOMEPAGE is unused (013 Q11).

## API (shared module `src/modules/content/shared/content-cate-sub/`, envelope `ContentEditResponseInterceptor`)
- Registry `SUB_CATE_TYPE_CONFIG`: ANIME (subTypeId 2, log `ANIME_CATE`), SPORT (subTypeId 1, log `TOURNAMENT_CATE`,
  `rootsOwnedByTourney: true`). Every query is scoped by SUB_TYPE_ID.
- Leaves with the same route set: `content/anime-cate` (menu `anime-cate`, old PUT :id / PUT update-status/:id are 409
  stubs) and `content/tournament-sub-cate` (menu `tournament-sub-cate`, `tournament` crontab on writes):
  `GET /` (?CATE_NAME, ?STATUS) · `GET :id` · `POST /` JSON `{ CATE_NAME*, PARENT_ID?, SORT_ORDER? (1–10000, default
  max+1), STATUS? (default 0), HIDDEN_DEVICE_LIST_V2? number[] }` · `PATCH :id` changed fields only · `PATCH :id/status
  { STATUS: 0|1 }` (whole subtree) · `PUT sort-order` · `DELETE :id` (subtree + links) · `GET vods/:id` · `DELETE vods/:id`.
  Read-only for clients: CATE_ID, CATE_CODE, CATE_NODE_TYPE, HIDDEN_DEVICE_LIST, SUB_TYPE_ID, TOURNEY_ID, SPORT_ID,
  GROUP_CATEGORY_ID, CATE_LINK, USER_*, dates.
- SPORT root locks (all 422): create with PARENT_ID 0, rename / move a root, move a child to PARENT_ID 0, delete a root.
  Root STATUS / SORT_ORDER / hidden devices stay editable. ("Root" = PARENT_ID 0.)
- Internal API used by the tournament module (inside its transaction): `createTourneyRoot`, `syncTourneyRoot`
  (CATE_NAME / SPORT_ID), `setStatus(…, manager?)`, `findTourneyRoots(ids | null)`, `countChildrenAndLinks`,
  `deleteTourneyRoot`. Their audit is part of the tournament's audit row (module `TOURNEY`).
- Excel CATE_LINK auto-link: `ContentCateSubLinksService.findCateIdsByCateLink(typeId, contentCateIds)` reads SPORT rows
  with CATE_LINK and parses the JSON in TS (CSV or array `cate_list`); `ContentCateLinkService.linkTourneysByCateLink`
  adds only missing links (used by `importExcelVod` and handle-excel inside its query runner).

## Consumers
- VOD / movie edit forms (008 field `TOURNAMENT_CATE_LIST`, name unchanged): `ContentCateLinkService` → sub-cate links for
  both trees (tournament TYPE_IDs 12, 78; Anime 2, 6, 20, 22, 25); roots and children selectable; `syncForContent`
  returns true when tournament links change (queue `tournament` crontab). VOD duplication copies CONTENT_CATE_SUB_LIST only.
- `lookup/cate-tree-options` and TPW menu "Cate phụ" (right column + left COUNT) read CONTENT_CATE_SUB for every
  registered sub type; unregistered → [].
- Web: module `src/modules/content/SubCate` (ex-AnimeCate) parameterised by type (`SUB_CATE_TYPES`): Anime at
  `/content/anime/category`, Giải đấu at `/content/sub-cate/tournament` (`?root=<CATE_ID>` filters to one tournament).

## Menus / permissions (stage applied 2026-10-20; prod with the release)
- "Danh mục phụ" (`anime`) ⊃ Anime (`anime-cate`), Giải đấu (`tournament-sub-cate`). Script + grants:
  `api/.ai/features/013-tourney-content-cate-sub/baseline/menu-013.js`.
