---
title: Menu export / import
tags:
  - contracts
  - api
  - web
  - permissions
summary: >-
  Export the cmssmart_admin_menu table to JSON and import it back (dry-run by
  default, never deletes, skips ambiguous duplicates). Used to sync menus
  between environments.
status: ready
links:
  - architecture/decisions/0004-permission-model.md
  - architecture/decisions/0001-new-feature-module-conventions.md
  - repos/api.md
  - repos/web.md
updated: '2026-10-28'
---
# Menu export / import

Api: `src/modules/administration/menu-transfer/` (plain classes per ADR 0001, own response interceptor, `@ApiExcludeController`). Web: `MenuManager/components/MenuTransferButtons.tsx` (buttons "Xuất menu" / "Nhập menu" in the Quản lý Menu action bar). Table: `cmssmart_admin_menu` (levels 1-4; level 4 = permissions whose `menu_key` is the `@RouteInfo` name).

## Endpoints (standard envelope, `data` = payload)

| Route | Permission | Notes |
|---|---|---|
| `GET administration/menu-transfer/export?levels=1,2` | `menu-config.export` | `{ version: 1, exported_at, count, duplicates[], items[] }` |
| `POST administration/menu-transfer/import` | `menu-config.import` | body `{ dry_run?, version?, items }`; **dry-run unless `dry_run === false` or `'false'`** |

Page `menu_key` is `menu-config`; the two permission names are NEW, so they need permission-sync plus role grants (ADR 0004).

## File format
- Item: `menu_key`, `menu_level`, `parent_key` (null at level 1, else `menu_key` of the parent one level up), `menu_name`, `menu_order`, `isactive`, and text columns `menu_link, redirect_link, icon, api_link, api_key, key_word, guide_url`. **No `menu_id`** — ids differ per environment, so rows are matched by (`menu_level`, `menu_key`) and parents by `parent_key`.
- Max 5000 items per import.

## Import rules
- Never deletes. Creates missing rows, updates changed columns / parent, counts the rest as `unchanged`.
- Report: `{ dry_run, summary{received, created, updated, unchanged, skipped}, created[], updated[]{changes}, skipped{invalid, ambiguous, parent_not_found} }`.
- **Duplicates**: the real DB has (level, key) pairs on several rows (35 groups seen 2026-10-28). Export lists them in `duplicates`; import skips every copy as `ambiguous` (in the file or in the DB, plus children of an ambiguous parent) because it cannot tell which row is meant. Fix the data by hand.
- Apply runs in one transaction (level-by-level batch insert, CASE batch UPDATE), guarded by an in-process write lock; each insert/update is audit-logged under module `menu-config.import`. Client errors are generic (503 read failure / 500 apply failure, data unchanged).

## Web flow
Pick file → client sends `dry_run: true` → modal shows counts and skipped rows → "Áp dụng" sends `dry_run: false` (inactive when nothing to create/update) → invalidates `MenuList` / `AllMenuList` tags.

## Tests
- API: jest `menu-transfer` dir (reconcile / service / controller specs).
- Web e2e `e2e/tests/menu-transfer.spec.ts`: export download, re-import of the exported file shows 0/0, ghost item preview, bad file, API dry-run default. **Never applies** (shared DB).
