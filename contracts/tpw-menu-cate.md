---
title: TPW menu cate config ("Cấu hình theo loại") — per MENU_CATE_TYPE storage
summary: >-
  `configs/vod-common/tpw-menu-cate/:menuID` (multipart POST, GET) stores a
  menu's cate config by TPW_MENU_CATE_TYPE. Type 12 "Tất cả theo nhóm cate phụ"
  writes one TPW_MENU_CATE_DATA_ALL row (GROUP_CATEGORY_ID required, SPORT_ID 0
  = all); type 13 "Data từ cấu hình truyền thông" behaves exactly like type 3
  "Mới nhất". Shared lookups `group-categories` / `sports` feed the selects.
tags:
  - contract
  - api
  - web
  - vod-common
  - tpw-menu
status: ready
links:
  - contracts/group-category.md
  - contracts/sports.md
  - contracts/content-cate-sub.md
updated: '2026-10-22'
---
# TPW menu cate config — `configs/vod-common/tpw-menu-cate`

Web: route `/configs/vod-common/tpw-menu/:platform/advanced?selectedMenuId=…` → "Cấu hình theo loại" (menu type
`cate`) → `MutiCateConfig`.

## Storage by TPW_MENU_CATE.MENU_CATE_TYPE (TPW_MENU_CATE_TYPE)
| Type | Name | Child table | Payload |
|---|---|---|---|
| 1 | Data từ các cate gốc | TPW_MENU_CATE_DATA_CONFIG TYPE 1 | `CATES[]` |
| 4 / 7 / 8 | data list | TPW_MENU_CATE_DATA_LIST | `CONTENTS[]` |
| 5 | banner | TPW_MENU_BANNER_DATA_CONFIG | `TYPE_ID`, `TYPE`, `CONTENT_SINGLE` |
| 9 | BI | TPW_MENU_CATE_DATA_BI | `DATA_BIs[]` (required) |
| 11 | Data từ các cate phụ | TPW_MENU_CATE_DATA_CONFIG TYPE 2 | `CATES[]` (required) |
| **12** | **Tất cả theo nhóm cate phụ** | **TPW_MENU_CATE_DATA_ALL** | **`GROUP_CATEGORY_ID` (required, >0), `SPORT_ID` (0 = Tất cả, default)** |
| 3, **13**, others | default branch | TPW_MENU_CATE_DATA_CONFIG TYPE 1, one row | `TYPE_GROUP_DATA_ID`, `CATE_ID` |

- Every save deletes all child rows of the menu (all tables above) then writes the current type's rows, in one
  transaction; DELETE config also clears TPW_MENU_CATE_DATA_ALL.
- **TPW_MENU_CATE_DATA_ALL** (ID, MENU_CATE_ID, GROUP_CATEGORY_ID, SPORT_ID; the TYPE column was dropped 2026-10-22):
  exactly one row per menu. SPORT_ID 0 means all sports; legacy rows with NULL read back as 0.
  Missing group → 400 "Vui lòng chọn nhóm loại danh mục" (FE checks first with the same text).
- GET returns `GROUP_CATEGORY_ID` (null if none) and `SPORT_ID` for type 12.
- Type 13 has no extra UI, same as type 3. Default URLs on first create (CATE_NODE_TYPE 2, empty URL fields) include
  types 3 and 13 → `api/v{1,2}/vod-common/contents?menu_id=…` (`DEFAULT_URL_RULES` in `tpw-menu-cate.const.ts`).

## Shared lookups — `configs/vod-common/shared`
- `GET group-categories` → `[{ ID, NAME, STATUS }]` ordered SORT_ORDER, ID.
- `GET sports` → `[{ SPORT_ID, SPORT_NAME, STATUS }]` ordered SPORT_ID.
- Both include hidden rows (STATUS 0) so saved selections still render; web labels them "(ẩn)". Web prepends the
  sport option "Tất cả" (value 0).
