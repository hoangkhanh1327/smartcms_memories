---
title: api-smart-cms — API-side repo
tags:
  - repo
  - api
  - nestjs
summary: >-
  Backend API for the CMS project. NestJS 10 + Fastify, multi-store. Large and
  inconsistent repo — new code follows ADR 0001, never copy the legacy patterns.
status: ready
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
  - contracts/ai-dubbing.md
  - contracts/content-edit.md
  - contracts/content-media-upload.md
updated: '2026-10-07'
---
# `api-smart-cms` — API-side

**Path**: `/home/khanhth/projects/admin2/api` · **Role**: backend REST API for the CMS project.
No frontend is served from this repo.

## Shape

- **NestJS 10** + **Fastify** transport. TypeScript throughout.
- Global route prefix from the required **`BASE_URL`** env var (`src/main.ts`); deployed as
  `api/v1`. The `APIPrefix.version` enum in `src/constant/common.const.ts` is dead code — it has
  no references, so changing it does nothing.
- **Multi-store**: MySQL via TypeORM across ~19 *named* connections (`DB_TYPE` enum), MongoDB
  (Mongoose), Redis (5 named instances), ClickHouse (read-only analytics).
- Auth: JWT Bearer, global `AuthGuard` via `APP_GUARD`. RBAC route names via the `@RouteInfo`
  decorator.
- Deploy: Docker + K8s manifests under `deploy/`, PM2 for non-K8s envs.
- ~2,400 routes. A CodeGraph index exists (`.codegraph/`) — use
  `codegraph query -p <repo> <term>` to locate things rather than grepping blind.
- **`jq` is not installed on this machine.** Use `node` or `/usr/bin/python3` for JSON work in
  scripts and hooks. `node` lives under an nvm version path that changes on upgrade; `python3` is
  at a stable `/usr/bin/python3`.

## Working in this repo — read this first

The repo is **large and inconsistent**; several conventions coexist. Do not infer the house style
from whichever file you open first.

- **New feature module** → follow
  [`architecture/decisions/0001-new-feature-module-conventions.md`](../architecture/decisions/0001-new-feature-module-conventions.md).
  No base repository, no base service, no `IXxx` interfaces or DI tokens, no `BaseController`.
  The owner's folder references are `src/modules/config/referral` and
  `src/modules/config/gamification`: parent `<feature>.module.ts` only aggregates, one leaf per
  resource with its own module, `shared/` for cross-leaf code.
- **Existing module** → keep its local style. Do not migrate legacy code to ADR 0001 on your own;
  the owner asked for it once — feature 009 re-shaped `content/{music-video,movie,vod}` into ADR
  0001 leaves (pure move, legacy `IXxx` tokens kept; see the section below).
- `.ai/core/*.md` in the repo documents the **legacy** conventions (`provenance: inferred`) and is
  tagged `[legacy-only]` where ADR 0001 overrides it.
- A `PreToolUse` hook (`.claude/hooks/memory-commit-reminder.sh`, wired in `.claude/settings.json`)
  reminds the agent to update this memory when new module files are staged for commit.
- **Type-checking a few files** without a full `tsc`: ts-jest type-checks every file a spec
  imports, so a throwaway spec that `await import()`s the touched files (with
  `jest.mock('@/config/config', ...)`) reports their TS errors. Run jest per directory with
  `--runInBand`, never the whole suite. `nest start --watch` only restarts the server when the
  compile has no errors — a rebuilt `dist/` with an old process start time means a type error.
- Local API needs the VPN to the staging DB / Redis / Mongo; without it `nest start` compiles but
  hangs retrying connections (`ETIMEDOUT`).

## Known issues

**Open:**

1. `ReferralCampaignController` declares `@Get(':id')` above `@Get('report')`. Nest matches in
   declaration order, so `GET /api/v1/config/referral/campaign/report` is captured by `detail()`
   with `id = "report"` → `Number("report")` is `NaN`. The report endpoint is unreachable despite
   appearing in Swagger. Fix is a two-line reorder; confirm no client depends on current behaviour.
2. `AllExceptionFilter` sets `error: exception` (serialises the raw exception to the client) and,
   for a plain `Error`, puts `exception.stack` into the client-visible `message` with a 500. This
   is why new services must throw `HttpException` subclasses.
3. Legacy content services return `Error` objects instead of throwing in many places, so callers'
   `if (!x)` checks never fire: VOD `ContentService.getContent`, Music `getDetailMusicVideo`
   (always returns an object) as used by `thirdparty/services/sync-content`; and
   `movie/movie-trailer/controllers/movie-trailer.controller.ts` (renamed from `.controler.ts` in
   feature 009) compares an un-awaited `getDetails` Promise to null.
4. The redis-api `security_code` is hardcoded in source (moving it to env is a separate feature).

**Resolved 2026-09-03:**

5. ~~Referral module carried an unwired Excel-export path plus unused transaction-count helpers.~~
   Removed: `ExportReferralCampaignDto`, `findForExport`, `getStatusCounts`,
   `countDistinctReferees`, `calculateConversionRate`, and dead `fs` / `renderExportExcelBorder`
   imports — along with the 9 tests that covered them (17 → 8 tests, all passing; typecheck clean).
   Note for future cleanups: those methods *had* test coverage, so a grep that excludes `*.spec.ts`
   under-reports what a removal will break.

## Documented feature contracts

| Module | Path in repo | Contract |
|---|---|---|
| **AI Dubbing** — AI voice-over / dubbing pipeline (video job → language task → preview review → multi-audio final output) | `src/modules/content/ai_dubling/` (folder spelled without the second `b`) | [`contracts/ai-dubbing.md`](../contracts/ai-dubbing.md) |
| **Content edit** — VOD / Movie / Music Video detail, create, PATCH, trailer/series sub-tables | `src/modules/content/{vod,movie,music-video}/` leaves + `src/modules/content/shared/content-edit/` | [`contracts/content-edit.md`](../contracts/content-edit.md) |
| **Content media upload** — signed per-slot image upload to CDN | `src/modules/content/shared/media-upload/` | [`contracts/content-media-upload.md`](../contracts/content-media-upload.md) |

Read the contract before building or changing the frontend screens for one of these — it records
the request/response shapes, the status machines and the known dead filters, which the controllers
alone do not tell you.

## Menu permissions (RBAC) and permission sync

Added 2026-09-25 by feature `006-route-permission-sync`.

- A **permission** is a `cmssmart_admin_menu` row with `menu_level = 4` whose `menu_key` equals a
  handler's `@RouteInfo({ name })`. Its `menu_parent` is the menu whose `menu_key` equals the
  controller's class-level `@RouteInfo({ menu_key })`. Groups store granted keys as a CSV in
  `admin_group.permissions`; `RouteNameInterceptor` only checks `permissions.includes(name)` —
  the parent is for the CMS tree, not for enforcement. Handlers without a method-level `name` are
  not permission-checked at all.
- **`POST {BASE_URL}/administration/permission-sync`** (module
  `src/modules/administration/permission-sync/`, ADR 0001 shape) scans every controller's
  `@RouteInfo` via `DiscoveryService` and reconciles level-4 rows:
  - body `{ "dry_run": boolean }` — **defaults to a dry run**; only `false` / `"false"` writes.
  - creates missing permissions and updates `menu_name` (from `desc`) / `menu_parent` of existing
    ones; **never deletes** — rows no longer in code are reported as `orphans`.
  - skips and reports: controller without `menu_key`, parent menu not found, ambiguous parent
    (duplicate `menu_key` in the DB, or one `name` under several `menu_key`s), `name` already used
    by a non-permission menu, duplicate level-4 rows.
  - writes in one transaction (one batched INSERT, one CASE UPDATE), audit-logged via
    `LogActionService` (`module: permission-sync.sync`); an in-process lock returns 409 for a
    concurrent write. Guarded by its own permission `permission-sync.sync` under `menu-config`.
- The old `GET /route` (`src/modules/routes/`, public, deleted every level-4 row) and
  `MenuConfigService.addMenuDynamic` / `deleteMenuDynamic` were removed.
- `@RouteInfo.path` feeds the permission row's `api_link` — keep legacy path strings when a
  handler moves to a new controller, or permission-sync will rewrite existing rows.

- **Not on Swagger** (2026-09-28): `PermissionSyncController` is `@ApiExcludeController()` — it is
  an internal admin tool. Call it through the Postman collection
  `postman/permission-sync.postman_collection.json` (login → dry run → real write; collection
  variables `baseUrl`, `username`, `password`, `accessToken`; never commit real credentials into it).

- Permission sync can be scoped: body `controllers?: string[]` (controller class names, 1–50). The scan still reads all metadata (in-memory, no DB), then keeps permissions declared by any listed controller, so shared `name`s still report `ambiguous` exactly like a full run. DB read narrows to rows whose `menu_key` is a scanned `name` or parent `menu_key`. Unknown controller ⇒ 400 before any DB access. Scoped runs return `orphans: null` (partial table read); report carries `controllers` (`null` = full run).

## Content modules: VOD / Movie / Music Video (features 008 + 009)

Since feature 009 (2026-10-07) each kind is an ADR 0001 aggregator over per-resource leaves; the
feature-008 edit code lives inside them and `content/content-edit/` no longer exists:

```
content/<kind>/                       # kind = vod | movie | music-video
  <kind>.module.ts                    # aggregator: imports + exports the leaves only; keeps the old
                                      # class name (VodModule / MovieModule / MusicVideoModule)
  <kind>.module.spec.ts               # asserts the aggregator / leaf shape from module metadata
  <kind>/                             # main resource, module class <Kind>CoreModule: legacy controller
                                      # + 008 edit controller, services/, <kind>-edit.mapper.ts, dto/
  <kind>-cate/  <kind>-series/  <kind>-trailer/   # legacy + 008 standard routes per resource
  <kind>-subtitle-ai/                 # vod and movie only
  shared/                             # <Kind>SharedModule: all TypeORM entities + the kind's
                                      # repositories (+ providers several leaves need), shared dto/
content/shared/media-upload/          # MediaUploadModule (imported by app.module)
content/shared/content-edit/          # pure 008 helpers: field catalogue, body pipe, mapper,
                                      # interceptor, timeouts, actor-links, contracts, dto/
```

- Pure move: routes, permission names, `IXxx` string tokens and service internals are unchanged
  (route table + per-handler `RouteInfo` names diffed identical before/after). VOD files keep
  their `content*` names.
- DI rules used: `<Kind>SharedModule` is the first import of every leaf so its tokens win like the
  old module-local providers did; a cross-module provider the old module declared locally sits in
  the leaf that uses it, or in `shared` when several leaves use it; external module imports keep
  the old order. The main leaf also declares the cross-module tokens the old module exported
  (tag, actor, bestcut, crontab, tpw, banner, tournament …), so outside importers of
  `VodModule` / `MovieModule` / `MusicVideoModule` still see them.
- Cycles: `MovieService` ↔ `MovieSeriesService` (`forwardRef` on both leaves). VOD core → series
  and music-video core → series are one-way.
- Side effects (008) compute on the merged row; cate list and actor links (+ `CONTENT_ACTOR` /
  `MOVIE_ACTOR`) are written inside the save transaction; auto-tags reuse the legacy
  `syncAutoTags`. Trailer/series controllers are thin delegates; their services translate legacy
  `Error`/object results into `HttpException`.
- Legacy `ContentService` / `MovieService` / `MusicVideoService` still serve lists, status,
  publish, sync and other immediate actions; their old create/update/detail handlers were removed.
  `channel.service` (live → VOD) still re-saves a VOD through `ContentService.createContent` —
  moving it to the PATCH service is a separate feature.
- The `'IContentService'` (VOD) and `'IMusicVideoService'` string tokens and their interface files
  are gone; inject the classes (`@Inject(forwardRef(() => ContentService))` where the module import
  is already `forwardRef`). `config-webapp` has its own unrelated `'IContentService'` (landing page).
- Legacy conventions kept: VOD and Movie store an inactive episode as `-1` (Music uses `0`).
- Media-upload HMAC key derivation still uses the label `content-media-upload` — do not rename it
  (would invalidate tokens issued before a deploy).
