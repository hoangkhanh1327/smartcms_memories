---
name: smartcms-api-jobs-database-deploy
description: >-
  api-smart-cms (admin2/api): background jobs (BullMQ + legacy Bull),
  database/migrations, deployment topology (pm2 envs, Docker, docker-compose),
  CI/CD, and utility scripts
metadata:
  type: reference
title: 'smartcms-api — jobs, database, deploy'
tags:
  - repos
  - smartcms-api
  - backend
  - nestjs
  - deploy
  - jobs
summary: >-
  Background jobs, DB/migrations, deploy topology and CI/CD for the real
  api-smart-cms repo (admin2/api)
status: ready
updated: '2026-08-18'
---
# smartcms-api — background jobs, database, deployment, CI/CD

Repo: `/home/khanhth/projects/admin2/api` (package name `api-smart-cms`). This is the REAL production
SmartCMS backend — do not confuse with `repos/api.md`, which documents an unrelated generic
`backend-template` scaffold repo. See sibling files under `repos/smartcms-api/` for the rest of this
repo's map (core infra, modules by domain).

## Background jobs — two parallel, non-interoperating systems

**BullMQ (`@nestjs/bullmq`) — `src/modules/job-queue/`**
- Single queue `publish-content-jobs-{env}` (env-suffixed, shared Redis across envs — namespaced
  deliberately to avoid cross-env job execution).
- `PublishContentJobProcessor` (concurrency 10) flips `MOVIE_STATUS`/`CONTENT_STATUS`/`SERIES_STATUS`
  to published for movies, VOD content, and music videos (three parallel schemas), with validation
  (poster present, category assigned, ≥1 active series/trailer, not expired). On success also writes
  to an external Redis via `src/utils/redis-api.ts` `baseRedis()` carrying a **hardcoded `security_code`
  value duplicated in 3 places** in `publish-content-job.processor.ts` — a credential embedded in app
  code, not config.
- `PublishContentJobService` (file literally named `publish-content-job.serivce.ts` — typo) creates
  delayed jobs keyed by `publish-content-{contentID}-{typeID}`, replacing any existing job with the
  same ID. `attempts: 0` configured despite backoff settings — no automatic retry actually happens.
  Throws if `publishDate` is in the past.
- **Every failure path is swallowed**: the whole handler body is wrapped in try/catch that only logs
  to a file — BullMQ's own retry/failed-job accounting never sees these as failures. Any
  dashboard/alerting built on queue failure counts will be blind to real failures.

**Legacy Bull (`@nestjs/bull`, different package from BullMQ) — `src/modules/crontab/`**
- No `@Cron()` decorators anywhere in the repo. "Cron" jobs are actually BullMQ **repeatable jobs**
  registered by `InitCrontab`, skipped entirely when `env === 'local'`.
- Two repeatable jobs every 1 minute on `handle_ai_subtitle_request_1`: `GET_UNHANDLED_REQUESTS`,
  `GET_CONVERTED_FILES`. `CrontabConsumer` pushes matches onto a second queue
  `queue_processor_request_file_1`.
- `ProcessRequestFileConsumer` does HLS→AAC conversion via ffmpeg, then uploads to a hardcoded remote
  AI-server path (`/data/ndloc_bk/app/input`), tracking a `CONVERT_STATUS` state machine
  (1=queued, 2=converted, 3=queued-for-upload, 4=uploaded, 5=error).
- Both consumers manually set a 30-day TTL on `bull:{queue}:{jobId}`/`:logs` Redis keys — Bull leaves
  job data around indefinitely otherwise.

**No distributed lock mechanism anywhere in `src/`** (no redlock/SETNX helper). Prod runs 6 pm2
cluster instances plus 1-minute repeatable jobs — correctness under concurrency relies solely on
queue/job-ID dedup logic, not an explicit lock. Worth checking before adding new cron-like jobs.

## Database / migrations

- `database/` at repo root has **one file**, `database/script` — 7 lines of ad-hoc raw SQL
  (`ALTER TABLE channel ADD COLUMN ...`), no naming convention, no runner. Not a real migration system.
- Actual TypeORM migrations live inside feature modules instead: `src/modules/content/comic/migration`
  and `src/modules/content/movie/migration`. No root `data-source.ts`/`ormconfig` found.
- The `"migration"` package.json script is **broken**: `"nest start --watch && ts-node
  ./node_modules/typeorm/cli.js migration:run"` — `nest start --watch` never exits, so the migration
  command can never run via this script as written.

## Deployment topology

5 pm2 configs, port comes from `NODE_PORT` env var in the matching `build:*` script, not from the
JSON itself:

| Env | pm2 file | app name | port | instances | server path |
|---|---|---|---|---|---|
| dev | `pm2.yaml` (referenced by `build:dev`, **file doesn't exist**) | — | — | — | — |
| staging | `pm2.stag.json` | `api-smart-cms` | 5070 | 1 | `/home/source_code/smartcms/api/` |
| staging2 | `pm2.stag2.json` | `api-smart-cms-2` | 5073 | 1 | `/home/source_code_2/smartcms/api/` |
| pilot | `pm2.pilot.json` | `api-smartcms-pilot` | 5070 | 1 | `/home/source_code/smartcms/api/` |
| prod | `pm2.prod.json` | `api-smartcms-prod` | 5071 | **6, cluster mode** | `/home/source_code/smartcms_prod/api/` |

Only prod is clustered/multi-instance. Dockerfile (`node:18.16.0-alpine`, 2-stage) sets
`TZ=Asia/Ho_Chi_Minh` and its `CMD` is `pm2-runtime start pm2.json --env production` —
**`pm2.json` does not exist** (only the env-suffixed files above do); the Dockerfile path looks stale
— actual deploy is rsync (`bash/*.sh`) + pm2 directly on the host, not this Docker image.
`docker-compose.yaml` is staging-specific (mounts `.env.staging`) despite having no env suffix, and
assumes Redis/MySQL run externally on the host (no DB/Redis service defined in compose).

**`deploy/` (k8s manifests: configmap/deployment/secrets) is for a DIFFERENT, unrelated microservice**
(`api-promotion`/`api-channel`, image refs at `gitlab.softhcmits.xyz:5050/b2c/...`) — looks copy-pasted
from a sibling repo and never cleaned up. `deploy/secrets/secret.yaml` contains **base64 (not
encrypted) DB/Redis/DRM credentials committed to git** for that other service — real exposure risk
even though it's not this app's own secrets.

## CI/CD (`.gitlab-ci.yml`)

Two stages only: `deploy`, `upload_code` — **no build/test/lint gate at all**, straight to
rsync-deploy:
- `deploy-dev` (branch `dev_bk`, auto): `bash/dev.sh`
- `deploy-stage-v2` (branch `stage_v2`, auto): `bash/stage_v2.sh`
- `deploy-stage` (branch `stage`, manual): `bash/stage.sh`
- `deploy-preonline` / `deploy-prod`: **entirely commented out** — pilot/prod have no active CI deploy
  path; presumably done by hand via `bash/preonline.sh`/`bash/prod.sh`.
- `upcode-hotfix` (branch `hotfix/*`, manual): zips via `bash/upcode_hotfix.sh` for out-of-band hotfix
  delivery.

`bash/*.sh` (7 scripts) are all rsync-based deploy scripts with drifted, inconsistent
`--exclude` lists and hardcoded destination paths — no shared structure between them.
`bash/stage_new.sh` has its rsync step commented out entirely.

## Postman collections (partial API contract reference only)

Only 2 collections exist — do not treat as full API coverage:
- `postman/comic-homepage.postman_collection.json` — `/content/comic-homepage` CRUD + layout/ordering.
- `postman/referral-campaign.postman_collection.json` — `/config/referral/campaign` CRUD + stats/export.

## Utility scripts

- `script/refactor_imports.js` (`yarn refactor:imports`): codemod converting relative imports to the
  `@/...` alias form; dry-run by default, `--write` to apply.

## Chaos / risk findings worth remembering before touching this area

- `deploy/` k8s manifests belong to a different service entirely (`api-promotion`) — don't assume
  they describe how this app is deployed, and treat `deploy/secrets/secret.yaml` as a leaked-secrets
  cleanup candidate regardless.
- Hardcoded `security_code` secret in `publish-content-job.processor.ts` (3 places) should move to config.
- Dockerfile and `build:dev`'s `pm2.yaml` both reference files that don't exist — likely dead/stale paths.
- `"migration"` npm script cannot work as written (see above).
- CI has zero automated test/lint/build gating — quality control is entirely manual/out-of-band.
- Two separate queueing stacks (BullMQ + legacy Bull) with different Redis namespaces double the
  operational surface for anyone debugging job/queue issues.
- Job failure handling swallows errors everywhere — don't trust BullMQ dashboards' failure counts as
  ground truth for this app without checking the log files directly.
- Filename typos to expect when searching: `publish-content-job.serivce.ts`,
  `deploy/deploymemt_new.yaml`.
