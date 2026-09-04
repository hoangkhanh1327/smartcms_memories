---
title: api-smart-cms — API-side repo
tags:
  - repo
  - api
  - nestjs
updated: '2026-09-04'
summary: >-
  Backend API for the CMS project. NestJS 10 + Fastify, multi-store. Large and
  inconsistent repo — new code follows ADR 0001, never copy the legacy patterns.
status: ready
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
  - contracts/ai-dubbing.md
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
- **Existing module** → keep its local style. Do not migrate legacy code to ADR 0001.
- `.ai/core/*.md` in the repo documents the **legacy** conventions (`provenance: inferred`) and is
  tagged `[legacy-only]` where ADR 0001 overrides it.
- A `PreToolUse` hook (`.claude/hooks/memory-commit-reminder.sh`, wired in `.claude/settings.json`)
  reminds the agent to update this memory when new module files are staged for commit.

## Known issues

**Open:**

1. `ReferralCampaignController` declares `@Get(':id')` above `@Get('report')`. Nest matches in
   declaration order, so `GET /api/v1/config/referral/campaign/report` is captured by `detail()`
   with `id = "report"` → `Number("report")` is `NaN`. The report endpoint is unreachable despite
   appearing in Swagger. Fix is a two-line reorder; confirm no client depends on current behaviour.
2. `AllExceptionFilter` sets `error: exception` (serialises the raw exception to the client) and,
   for a plain `Error`, puts `exception.stack` into the client-visible `message` with a 500. This
   is why new services must throw `HttpException` subclasses.

**Resolved 2026-09-03:**

3. ~~Referral module carried an unwired Excel-export path plus unused transaction-count helpers.~~
   Removed: `ExportReferralCampaignDto`, `findForExport`, `getStatusCounts`,
   `countDistinctReferees`, `calculateConversionRate`, and dead `fs` / `renderExportExcelBorder`
   imports — along with the 9 tests that covered them (17 → 8 tests, all passing; typecheck clean).
   Note for future cleanups: those methods *had* test coverage, so a grep that excludes `*.spec.ts`
   under-reports what a removal will break.

## Documented feature contracts

| Module | Path in repo | Contract |
|---|---|---|
| **AI Dubbing** — AI voice-over / dubbing pipeline (video job → language task → preview review → multi-audio final output) | `src/modules/content/ai_dubling/` (folder spelled without the second `b`) | [`contracts/ai-dubbing.md`](../contracts/ai-dubbing.md) |

Read the contract before building or changing the frontend screens for one of these — it records
the request/response shapes, the status machines and the known dead filters, which the controllers
alone do not tell you.
