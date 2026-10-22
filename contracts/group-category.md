---
title: Category groups (GROUP_CATEGORY) and their categories (GROUP_CATEGORY_CATE)
summary: >-
  Feature 014: "Nhóm loại danh mục" is GROUP_CATEGORY (menu/permissions
  `group-category.*` under Sporthub, TOURNEY-like shape with raw-JSON
  URL_GET_DATA) with categories in GROUP_CATEGORY_CATE_LIST; "Thể loại nhóm danh
  mục" is GROUP_CATEGORY_CATE (TYPE_ID 97). Delete blocked while
  CONTENT_CATE_SUB / TPW_MENU_CATE_DATA_ALL reference the group.
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
  - contracts/content-cate-sub.md
  - contracts/tpw-menu-cate.md
updated: '2026-10-22'
---
# Category groups — feature 014

## Storage
- `GROUP_CATEGORY`: ID, NAME, LOGO_TV / LOGO_TV_SMALL / LOGO_PHONE, SORT_ORDER, STATUS (**0 ẩn / 1 hiện** — the column
  comment copied from TOURNEY is wrong), URL_GET_DATA (JSON like TPW_MENU_CATE.URL_GET_DATA, ≤1000), CREATED_AT,
  MODIFIED_AT, USER_ID, USER_NAME.
- `GROUP_CATEGORY_CATE_LIST`: (CONTENT_ID = ID, CATE_ID). `GROUP_CATEGORY_CATE`: TYPE_ID fixed **97**; VOD Common TPW
  group 22 reads it.
- Referenced by `CONTENT_CATE_SUB.GROUP_CATEGORY_ID` (**column default 1** → group 1 is effectively undeletable; the
  Giải đấu roots carry 0; CMS never writes this column) and `TPW_MENU_CATE_DATA_ALL.GROUP_CATEGORY_ID` (written by the
  TPW menu cate config, menu type 12 — see contracts/tpw-menu-cate.md).

## API — `content/group-category` (module `src/modules/content/group-category/`, menu `group-category`)
- `GET /` (?NAME, ?STATUS) ordered SORT_ORDER, ID; rows + `GROUP_CATEGORY_CATE_IDS`, `GROUP_CATEGORY_CATES` · `GET :id`
  (row + categories read in parallel). p99 < 75 ms on stage.
- `POST /` JSON `{ NAME* (unique → 422), LOGO_PHONE*, LOGO_TV*, LOGO_TV_SMALL*, SORT_ORDER? (default max+1, 1–10000),
  STATUS? (default 1), URL_GET_DATA? (string parsing to a JSON object → else 400 "Trường URL_GET_DATA phải là JSON hợp
  lệ"; null clears), GROUP_CATEGORY_CATE_IDS?, MEDIA_TOKENS }` · `PATCH :id` changed fields only.
- `PATCH :id/status { STATUS: 0|1 }` · `DELETE :id` → 422 "Nhóm loại danh mục đang được dùng (danh mục phụ: n, …),
  không thể xoá", else tx: list rows → group.
- Logos: `POST content/media-upload/group-category/{create,update}` (kind GROUP_CATEGORY, same sizes as TOURNEY).
- Audit module `GROUP_CATEGORY`. No crontab cache keyword.

## API — `content/group-category-cate` (menu `group-category-cate`, "Thể loại nhóm danh mục") and lookup
- Fixed-type content-cate kind `GROUP_CATEGORY` (same routes as sports-cate, no delete).
  `GET lookup/group-category-cates` (JWT only).

## Web
- `VOD/GroupCategory` (`/content/group-category`): URL_GET_DATA as a monospace textarea with client JSON check.
- `VOD/FixedTypeCate` kind GROUP_CATEGORY (`/content/group-category-cate`).

## Change 2026-10-22 — "Thể loại nhóm danh mục" leaves the menu
- Opened by the "Thể loại" button on `/content/group-category`; `GroupCategoryCateController` `@RouteInfo` `menu_key` is
  now `group-category`; `group-category-cate.*` level-4 rows moved under it and the `group-category-cate` level-3 row is
  inactive. Full details: contracts/sports.md "Change 2026-10-22".
