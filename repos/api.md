---
name: backend-template-overview
description: >-
  Structure and conventions of the NestJS backend-template repo (Modular Clean
  Architecture / DDD)
metadata:
  type: reference
title: api
tags:
  - repos
summary: '# backend-template — structure overview'
status: ready
updated: '2026-08-18'
---
# backend-template — structure overview

NestJS (Fastify adapter) backend template, v2.0.0, described as "Modular Clean Architecture DDD Backend Template". Location: `~/projects/templates/backend-template`.

## Tech stack
- Runtime: NestJS 10 on `@nestjs/platform-fastify` (not Express)
- DBs: TypeORM (Postgres/MySQL via `pg`/`mysql2`), Mongoose (MongoDB), `@clickhouse/client` (ClickHouse)
- Cache/Queue: ioredis + BullMQ (`@nestjs/bullmq`)
- Auth: `@nestjs/passport` + `passport-jwt`, JWT access/refresh tokens
- Logging: `nestjs-pino` (Pino), custom `PinoLoggerService`
- Validation: `class-validator` / `class-transformer`, global `ValidationPipe` (whitelist + forbidNonWhitelisted + transform)
- Docs: `@nestjs/swagger`, served at `/docs` behind Basic Auth (`USERNAME_SWAGGER`/`PASSWORD_SWAGGER` env, default admin/admin123)
- Global API prefix: `api/v1`
- Misc: sharp (images), exceljs (excel), ssh2-sftp-client/basic-ftp (file transfer), fastify-multipart/fastify-multer (uploads)

## Top-level `src/` layout
```
src/
  main.ts            bootstrap: Fastify adapter, global prefix, pipes, interceptors, filters, Swagger
  app.module.ts       root module — imports CoreModule, SharedModule, feature modules; wires TraceIdMiddleware globally
  config/             plain config factory files (app.config.ts, database.config.ts) for @nestjs/config
  core/               cross-cutting infrastructure (see below)
  common/             legacy home for filters/guards/decorators — being migrated into core/ (see note)
  shared/             reusable app-level building blocks shared across feature modules
  modules/            feature modules (DDD-style: controller + service + module + dto/ + entities/)
  utils/              (currently empty/placeholder)
```

## `src/core/` — infrastructure layer
One subfolder per concern, each typically with its own `.module.ts`:
- `config/` — `env.schema.ts` (env validation schema)
- `context/` — `als.context.ts` (AsyncLocalStorage request context), `trace-id.middleware.ts`, `mac-address.middleware.ts`
- `database/` — `database.module.ts` (TypeORM), `mongo.module.ts`, `clickhouse.module.ts`, `data-source.ts` (TypeORM CLI data source for migrations), `transaction.decorator.ts`
- `guards/` — JWT strategy (`jwt-access.strategy.ts`), `jwt-auth.guard.ts`, `roles.guard.ts`/`roles.decorator.ts`, `permissions.guard.ts`/`permissions.decorator.ts`, `public.decorator.ts` (bypass auth), barrel `index.ts`
- `filters/` — `all-exceptions.filter.ts` (registered globally in main.ts)
- `interceptors/` — `logging.interceptor.ts`, `transform.interceptor.ts` (both registered globally)
- `logger/` — `logger.module.ts`, `logger.service.ts` (Pino wrapper, `PinoLoggerService`)
- `redis/`, `queue/` (BullMQ), `scheduler/` (`cron-lock.service.ts` — distributed cron locking), `http/` (outbound `HttpClientService`, `solr-sync.middleware.ts`), `cdn/`, `storage/` (providers/services/pipes/decorators/interfaces — file storage abstraction), `swagger/` (`swagger-auth.middleware.ts`)
- `core.module.ts` — aggregates the above into one module imported once from `AppModule`

## `src/shared/` — shared app-level utilities
- `shared.module.ts` — aggregator imported into `AppModule`
- `decorators/` — `current-user.decorator.ts`, `client-ip.decorator.ts`, `user-agent.decorator.ts`, `match.decorator.ts` (e.g. password-confirmation validation)
- `dtos/` — `pagination.dto.ts`, `api-response.dto.ts` (standard response envelope)
- `constants/` — `tokens.constant.ts`, `events.constant.ts` (event-emitter event names), `queues.constant.ts` (BullMQ queue names)
- `excel/`, `media/` (ffmpeg service), `utils/string.util.ts`

## `src/common/`
Only `filters/http-exception.filter.ts` remains. Per commit `3352169` ("consolidate DTOs and relocate guards, strategies, decorators to core"), this folder is being phased out in favor of `core/` — new cross-cutting code should go in `core/`, not `common/`.

## `src/modules/` — feature modules (DDD-ish per-module structure)
Each module folder generally contains: `<name>.module.ts`, `<name>.controller.ts`, `<name>.service.ts`, `dto/`, `entities/`, optionally `listeners/` (event-emitter handlers), `decorators/`, `interceptors/`.
- `auth/` — login, JWT access+refresh token issuance (`auth.service.ts`, `token.service.ts`), `RefreshToken` entity
- `users/` — CRUD, emits user events consumed by `listeners/user-events.listener.ts`
- `audit-logs/` — cross-module audit trail; exposes `@AuditLog()` decorator + `AuditInterceptor` for other modules to opt into logging, plus its own listener/entity/controller
- `job-queue/` — BullMQ integration; `publish-content/` subfolder shows the pattern: controller (enqueue) + processor (BullMQ worker) + service + dto
- `orders/` — present on disk but **not imported into `AppModule`** — looks like an example/scratch module, not wired into the running app

Only `CoreModule, SharedModule, AuthModule, UserModule, AuditLogModule, JobQueueModule` are actually registered in `app.module.ts`.

## Conventions worth knowing
- Path alias `@/*` → `src/*` (see jest `moduleNameMapper` and tsconfig)
- Auth: routes are protected by default; use `@Public()` to bypass, `@Roles()`/`@Permissions()` + their guards for authorization
- Standardized response shape via global `TransformInterceptor` + `shared/dtos/api-response.dto.ts`
- Migrations via TypeORM CLI: `yarn migration:generate|run|revert`, data source at `src/core/database/data-source.ts`
