---
title: Sports (SPORTS) and sports categories (SPORTS_CATE)
summary: >-
  Feature 014: "Môn thể thao" is SPORTS (menu/permissions `sports.*` under
  Sporthub) with categories in SPORTS_CATE_LIST; "Thể loại môn thể thao" is
  SPORTS_CATE (TYPE_ID 96) as a fixed-type content-cate kind. Delete is blocked
  (422) while TOURNEY / CONTENT_CATE_SUB / HTS / TPW_MENU_CATE_DATA_ALL
  reference the sport.
tags:
  - contract
  - api
  - web
  - content
  - sports
status: ready
links:
  - contracts/tourney.md
  - contracts/content-cate.md
  - contracts/content-media-upload.md
updated: '2026-10-22'
---
# Sports — feature 014

## Storage
- `SPORTS`: SPORT_ID, SPORT_NAME (NOT NULL), STATUS (0 ẩn / 1 hiện), AVATAR (NOT NULL — `''` when none), SLUG (page-info
  web, nullable), CREATE_DATE, MODIFY_DATE, USERNAME (no USER_ID column). One entity (`content/sports/entities`) also
  serves `GET lookup/sports` (active rows only).
- `SPORTS_CATE_LIST`: (CONTENT_ID = SPORT_ID, CATE_ID) composite PK. `SPORTS_CATE`: TOURNEY_CATE column convention,
  TYPE_ID fixed **96**; VOD Common TPW group 21 reads it.
- Referenced by: `TOURNEY.SPORT_ID`, `CONTENT_CATE_SUB.SPORT_ID`, `HTS.SPORT_ID`, `TPW_MENU_CATE_DATA_ALL.SPORT_ID` (no FKs).

## API — `content/sports` (module `src/modules/content/sports/`, menu `sports`)
- `GET /` (?SPORT_NAME LIKE, ?STATUS 0|1) rows + `SPORTS_CATE_IDS`, `SPORTS_CATES` (two parallel reads) · `GET :id`.
- `POST /` JSON `{ SPORT_NAME* (≤255, unique case-insensitive → 422), AVATAR?, SLUG?, STATUS? (default 1),
  SPORTS_CATE_IDS? (TYPE_ID 96 → else 422), MEDIA_TOKENS }`. SLUG not sent → server generates from the name (unaccented
  kebab-case, `-2`, `-3`… on collision); a sent SLUG that is taken → 422 "Slug đã tồn tại"; pattern
  `^[a-z0-9]+(-[a-z0-9]+)*$`.
- `PATCH :id` changed fields only; **rename never regenerates the slug**; `AVATAR: null` stores `''`; categories
  diff-synced in the same tx.
- `PATCH :id/status { STATUS: 0|1 }` · `DELETE :id` → 422 "Môn thể thao đang được dùng (giải đấu: n, …), không thể
  xoá" (only non-zero places listed; count inside the delete tx), else tx: SPORTS_CATE_LIST → SPORTS.
- Avatar: `POST content/media-upload/sports/{create,update}` (kind SPORTS, slot AVATAR, no fixed size).
- Audit module `SPORTS`. No crontab cache keyword exists for sports (checked CRONTAB_TYPE 2026-10-21).

## API — `content/sports-cate` (menu `sports-cate`) and lookup
- Fixed-type content-cate kind `SPORTS`: list / detail / POST / PATCH / `PATCH :id/status` / `PUT sort-order`; no
  delete. `GET lookup/sports-cates` → `{ value, label, parentId, nodeType }` (JWT only).

## Web
- `VOD/Sports` (`/content/sports`): slug auto-fills from the name **on create only** until edited, "Tạo lại từ tên"
  button; create omits SLUG when untouched (server generates).
- `VOD/FixedTypeCate` kind SPORTS (`/content/sports-cate`); tree picker `FixedTypeCateTreeField kind="SPORTS"`.
- `SportSelect` (Giải đấu form) shows a selected-but-hidden sport as "Môn #id (đã ẩn)".

## Change 2026-10-22 — category screens leave the menu (all three fixed-type kinds)
- "Thể loại môn thể thao" / "Thể loại giải đấu" / "Thể loại nhóm danh mục" no longer have a sidebar item. They open from a
  **"Thể loại"** button on the parent list page (`/content/sports`, `/content/tournament`, `/content/group-category`);
  the category list has a back button to the parent (`FIXED_TYPE_CATE_SCREENS[kind].parent`,
  `FixedTypeCate/components/FixedTypeCateLinkButton.tsx`). Routes `/content/*-cate` are unchanged.
- Permission names are unchanged (`sports-cate.*`, `tourney-cate.*`, `group-category-cate.*`), but the controllers'
  `@RouteInfo` `menu_key` is now the parent (`sports`, `tournament`, `group-category`), so permission-sync keeps the
  level-4 rows under the parent and the role editor shows them there.
- DB: level-3 rows `sports-cate` / `tourney-cate` / `group-category-cate` are `isactive = 0` (not deleted), and their
  level-4 rows are re-parented — script `.ai/features/014-sports-group-category/baseline/menu-cate-buttons.js`
  (preview / apply / revert; matched by menu_key). Applied on stage 2026-10-22; **prod must run `apply` in the same
  release as the API change.**
- Why not just `isactive = 0`: the role editor (`GET /administration/menu`, active rows only, tree by menu_parent)
  drops level-4 rows whose parent is inactive, so the permissions could no longer be granted.
- Side effects: no sidebar item is highlighted on the category screens, and the "HDSD" button (needs the menu row's
  `guide_url`) does not show there.

## Correction 2026-10-22 — no menu scripts for prod
- `menu-014.js` and `menu-cate-buttons.js` were deleted (owner decision; feature 014 was not on prod yet). Prod menus,
  permission rows and role grants are added **by hand** at release. Target state: sidebar items `sports` and
  `group-category` under Sporthub; no items for the three category screens; their level-4 permissions under
  `tournament` / `sports` / `group-category` (= controllers' `@RouteInfo` `menu_key`, so permission-sync places them
  there once the parent rows exist). Stage already matches this state.
