---
title: 'ADR 0004 — Permission model, naming and rollout checklist'
tags:
  - conventions
  - architecture
  - permissions
  - agent-rule
summary: >-
  CROSS-REPO. How CMS permissions are modelled (group → page → level-4
  permission), how to name and declare them (@RouteInfo, action-level, immutable
  names), how they are checked at runtime, and the step-by-step checklist to
  ship a screen's permissions safely. Derived from features 005/006 and the AI
  dubbing rollout (2026-09-29).
status: draft
links:
  - conventions.md
  - repos/api.md
  - repos/web.md
  - contracts/ai-dubbing.md
updated: '2026-09-29'
---
# ADR 0004 — Permission model, naming and rollout checklist

**Scope: cross-repo** (API `api-smart-cms` + web `smartcms`). Status: draft until the developer
confirms; the AI dubbing module (2026-09-29) is the reference implementation.

## 1. Model

`cmssmart_admin_menu` holds three kinds of rows:

| Kind | Level | Rule |
|---|---|---|
| **Group** | 1 (sometimes 2) | Only groups navigation. No permission hangs off it. |
| **Page** | 2 or 3 | Has a `menu_link` (FE path) and **no level 1–3 children**. A level-2 page with no children renders as a direct sidebar link; level-3 rows render as tabs. |
| **Permission** | 4 | `menu_key` = a handler's `@RouteInfo({ name })`. **Always a direct child of a page.** Never rendered by the FE. |

A group's grants are the CSV `admin_group.permissions` (menu_keys of pages, groups and
permissions). Group `group_id = 1` (admin) bypasses every check.

Two independent effects of that CSV (known gap, see §6):
- **API access** — decided only by the level-4 `name` (§4).
- **Menu visibility** — `GET /administration/list-menu` returns a level-1–3 row only if its own
  `menu_key` is in the CSV, and the FE drops a child whose parent is missing. So a user must be
  granted the page **and every ancestor** to see it.

## 2. Rules for declaring permissions (API)

- **R1 — Action-level, not endpoint-level.** One permission per thing a user does on the screen,
  not one per endpoint. Many handlers share one `name`; the scanner merges them.
  - `<page>.view` covers **every read endpoint the screen needs** (list, detail, nested lists,
    previews, status/step logs used to support users).
  - Workflow steps performed by the same role share one permission (AI dubbing: create job, add
    language, approve, reject → `ai-dubbing.create`). Split only when the business separates roles.
  - Vocabulary: `view`, `create`, `update`, `delete`, `retry`, `publish`, `export`, `import`,
    `approve` (only when approval is a separate role). Add a new verb only when none fits.
- **R2 — Name format `<page-menu_key>.<action>`**, e.g. `ai-dubbing.view`. Page/group menu_keys
  contain no dots (the CMS menu form enforces `/^[\w-]+$/`); permission names always contain one,
  so they never collide (`key_conflict`).
- **R3 — Class `@RouteInfo({ menu_key })` = the page's menu_key.** Every controller that serves a
  screen uses that screen's page key, whatever its URL path or resource. Do not invent a menu_key
  per controller (AI dubbing had 4 non-existent keys → 19 permissions `parent_not_found`).
- **R4 — A granted `name` is immutable.** Renaming or removing it revokes it from every group at
  once (403). Rename only before any real group is granted, or ship a re-grant plan with the change.
  Lock each module's names with a contract spec (reference:
  `src/modules/content/ai_dubling/controllers/ai-dubbing-permissions.spec.ts`).
- **R5 — Every CMS-callable handler has a `name`.** A handler without one is **unchecked** (any
  logged-in user). Exceptions: `@Public()` machine endpoints (e.g. AI server callback). Adding a
  `name` to an existing unchecked handler is a **breaking change**: backfill the new key into the
  groups that already use the screen *before* deploying.
- **R6 — Shared lookups are not owned by one screen.** Endpoints reused across screens (content
  search, movie series, option lists) must not be guarded by one screen's permission — other screens
  would break. Keep them authenticated-only, or give the consuming screen its own endpoint. Before
  adding a `name` to any endpoint, grep the FE for every screen that calls it.
- **R7 — `desc` is the Vietnamese display name** shown on the CMS. Give every handler sharing a
  `name` the same `desc` (the scanner keeps the first non-empty one).

## 3. Checklist — shipping a screen's permissions

1. **CMS menus**: create the group (if new) and the page (`menu_key` without dots, `menu_link` =
   FE route). Permissions are NOT created by hand.
2. **Code**: class `@RouteInfo({ menu_key: '<page>' })` on every controller of the screen;
   `@RouteInfo({ name: '<page>.<action>', desc, path })` on every handler.
3. **Contract spec** mapping each handler to its permission (R4).
4. **Dry run, scoped**: `POST {BASE_URL}/administration/permission-sync`
   `{ "dry_run": true, "controllers": ["XController", ...] }` → expect `created = N`, `skipped = 0`.
   Fix every `skipped` entry (§5) before writing.
5. **Write** with `"dry_run": false`, then dry-run again → `created = 0`, `updated = 0`.
6. **Clean up** rows from old names: they appear as `orphans` only on a full (unscoped) run; the
   sync never deletes — remove them on the CMS.
7. **Grant**: tick the page, its ancestors and the permissions; then **verify the saved CSV in the
   DB** (the group tree has a known bug, §6).
8. **Test matrix** with a non-admin group: view-only → reads 200, every action 403; add each action
   permission and re-test; revoke one → 403 immediately (cache is invalidated in-process; other
   instances within 60 s).

## 4. How a request is checked (API runtime)

1. `AuthGuard` verifies the JWT and resolves the user state (in-process cache, no Redis):
   `rejected` → 401; otherwise sets `request.user` / `request.authState`.
2. `RouteNameInterceptor` reads the handler's `@RouteInfo({ name })`:
   - no `name` → pass (unchecked, R5);
   - admin group → pass;
   - `name` in the group's permissions → pass; else **403**;
   - user state `unavailable` (DB down) → admin passes, others **503** (never a logout).
3. Menu `menu_key` / `menu_parent` play **no role** in the check — only `name`.

FE today: no route guard, no button gating; the sidebar comes from `list-menu` (levels ≤ 3). Active
menu is resolved by `menu_link` prefix match, so detail routes must live under the page's
`menu_link` (e.g. `/ai/dubbing/:id` under `/ai/dubbing`).

## 5. Sync skip reasons → fix

| `skipped.*` | Meaning | Fix |
|---|---|---|
| `no_menu_key` | Controller lacks class `@RouteInfo({ menu_key })` | Add the page key (R3) |
| `parent_not_found` | No level 1–3 menu has that key | Create the page on the CMS, or point the class to the right page key |
| `ambiguous` / `multiple_menu_keys` | Same `name` declared under different page keys | Make the controllers share one page key, or rename |
| `ambiguous` / `duplicate_parent` | Two menus share the page key | Rename the extra menu (`candidates` lists ids) |
| `ambiguous` / `duplicate_permission` | Two level-4 rows share the name | Delete the duplicate on the CMS |
| `key_conflict` | A level 1–3 menu uses the permission's name as its key | Rename that menu key |

## 6. Known gaps (planned for feature 007)

- Visibility is not derived: granting `x.view` without the page/ancestor keys hides the menu.
  Target: the API derives page + ancestors from granted level-4 keys.
- Web group-permission tree (`Group/components/Selects/GroupSelect.tsx`) passes the flat menu list
  to Kendo `handleTreeViewCheckChange`, so ticking/unticking a non-root node adds/removes an
  unrelated key. Keys absent from the tree are preserved. Fix: pass the rendered hierarchical data.
- `RoleService.update/create` write no audit log, so past CSV changes cannot be traced.
- No FE route guard or per-button gating; target is route `menuKey` + permission list from the API.
