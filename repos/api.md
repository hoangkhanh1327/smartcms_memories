---
title: api-smart-cms — API-side repo
tags:
  - repo
  - api
  - nestjs
updated: '2026-09-03'
summary: >-
  Backend API cho CMS project. NestJS 10 + Fastify, multi-DB. Repo lớn và messy
  — new code theo ADR 0001, không copy legacy.
status: ready
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
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

## Working in this repo — read this first

The repo is **large and inconsistent**; several conventions coexist. Do not infer the house style
from whichever file you open first.

- **New feature module** → follow
  [`architecture/decisions/0001-new-feature-module-conventions.md`](../architecture/decisions/0001-new-feature-module-conventions.md).
  No base repository, no base service, no `IXxx` interfaces or DI tokens, no `BaseController`.
- **Existing module** → keep its local style. Do not migrate legacy code to ADR 0001.
- `.ai/core/*.md` in the repo documents the **legacy** conventions (`provenance: inferred`) and is
  tagged `[legacy-only]` where ADR 0001 overrides it.

## Known issues (open, not fixed)

1. `ReferralCampaignController` declares `@Get(':id')` above `@Get('report')`. Nest matches in
   declaration order, so `GET /api/v1/config/referral/campaign/report` is captured by `detail()`
   with `id = "report"` → `Number("report")` is `NaN`. The report endpoint is unreachable despite
   appearing in Swagger.
2. `AllExceptionFilter` sets `error: exception` (serialises the raw exception to the client) and,
   for a plain `Error`, puts `exception.stack` into the client-visible `message` with a 500. This
   is why new services must throw `HttpException` subclasses.
3. The referral module carries a fully-built but unwired Excel-export path
   (`ExportReferralCampaignDto`, `findForExport`, `renderExportExcelBorder` import) plus unused
   `getStatusCounts` / `countDistinctReferees` / `calculateConversionRate` — dead code, no callers.
