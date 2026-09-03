---
title: ADR 0001 — Conventions for NEW feature modules
tags:
  - conventions
  - architecture
  - nestjs
  - agent-rule
updated: '2026-09-03'
summary: >-
  New feature modules must NOT follow legacy patterns: no base-repository, no
  base-service, no interface+DI-token, no BaseController. Defines the required
  folder shape and the response-envelope path.
status: ready
links:
  - conventions.md
---
# ADR 0001 — Conventions for NEW feature modules

**Status**: accepted · **Date**: 2026-09-03 · **Decided by**: project owner

## Scope — read this first

Applies to **new feature modules only**. Existing modules keep their current patterns; do not
migrate them, and do not "fix" a legacy module to match this ADR. The two styles coexist
deliberately.

A file added *inside* an existing legacy module follows that module's local style, not this ADR.
This ADR governs whole new feature modules.

## Decision

New feature modules deliberately **break** from the legacy conventions still documented in
`.ai/core/project-rules.md`. That file is `provenance: inferred` — it *describes* the old code,
it does not mandate it for new code.

### Do NOT use

| Legacy pattern | Why it's dropped for new code |
|---|---|
| `BaseRepository<T>` / `base-v2.repository.ts` | New repositories are plain `@Injectable()` classes that inject TypeORM's `Repository<T>` directly. No shared base class. |
| A base/abstract service layer | Services are plain `@Injectable()` classes. No base class, no shared abstract parent. |
| `IXxxRepository` / `IXxxService` interfaces + string DI tokens (`@Inject('IMovieRepository')`) | Inject the **concrete class** directly. No interface file, no token indirection. The abstraction was never actually used to swap implementations. |
| `BaseController` + `sendOkResponse` / `sendFailedResponse` | New controllers return **plain values** and get the envelope from a per-module interceptor — see "Response envelope" below. |

### Folder structure — multi-level module

Canonical shape (the in-repo reference is `src/modules/config/referral/referral-campaign`):

```
src/modules/<domain>/<feature>/
  <feature>.module.ts            # thin aggregator: imports + re-exports the leaf modules only,
                                 # no controllers/providers of its own
  <leaf>/
    dto/<leaf>.dto.ts            # several request DTO classes may share one file
    entities/<leaf>.entity.ts    # primary entity
    entities/<satellite>.entity.ts
    <leaf>.controller.ts
    <leaf>.module.ts
    <leaf>.repository.ts
    <satellite>.repository.ts    # satellite entity gets its own repo, no controller/service
    <leaf>.service.ts
```

Rules that come with the shape:
- `dto/` and `entities/` stay as subdirectories. Everything else is **flat** at the leaf root —
  do not create `services/`, `repositories/`, or `controllers/` folders for a single file.
- A **satellite entity** (a second table owned by the same feature) gets its own
  `<satellite>.repository.ts` but **no** controller or service of its own. It is read and written
  only through the primary entity's service.
- The parent `<feature>.module.ts` is an aggregator: `imports` + `exports` the leaves, with empty
  `providers` and `controllers`.

### Response envelope — per-module interceptor

The global `TransformInterceptor` (`src/main.ts`, `app.useGlobalInterceptors`) **cannot** build an
envelope from a raw return value. It expects an already-enveloped object and only re-maps its
fields. Returning a plain object from a controller with no other handling yields
`response.status(undefined)` and `data: undefined`.

Decision: new modules declare **their own interceptor** and apply it with `@UseInterceptors(...)`
on the controller. The global interceptor stays untouched.

**This composes only if the field names line up.** Nest runs global interceptors *outermost*, so
on the response path the module interceptor maps first, then the global one. The module
interceptor must therefore emit the **internal `BaseResponse` shape**, using `errors` (plural):

```ts
// module interceptor output — consumed by the global TransformInterceptor
{ status, result, message, data, errors }
                              // ^^^^^^ plural. The global interceptor reads `data.errors`
                              // and re-maps it to the wire field `error` (singular).
```

Emitting `error` (singular) from the module interceptor silently produces `error: null` on the
wire. `status` must always be set — the global interceptor calls `response.status(data?.status)`.

Final wire shape is unchanged from legacy endpoints:

```jsonc
{ "status": 200, "result": 0, "message": "success", "data": {}, "error": null }
```

### Error handling — throw typed exceptions

New services **must throw Nest `HttpException` subclasses** (`BadRequestException`,
`NotFoundException`, …) — never a plain `new Error('...')`, and controllers must not wrap every
handler in `try/catch`.

Reason: `AllExceptionFilter` already emits the correct envelope and honours
`exception.getStatus()`. But for a plain `Error` it falls through to a **500 and puts
`exception.stack` in the client-visible `message`**. Legacy controllers hide this by catching in
the controller; new controllers, which don't catch, would leak stack traces. Throwing a typed
exception avoids that path entirely.

> Known pre-existing issue, not introduced here: `AllExceptionFilter` also sets `error: exception`,
> serialising the raw exception to the client. Worth fixing separately.

## What still applies to new modules

Dropping the patterns above does **not** exempt new code from the ratified constitution or the
cross-cutting rules:

1. **No DB access from controllers.** Controller → Service → Repository stays. Only the base
   classes and interfaces are gone, not the layering.
2. **Audit log on every mutation.** Inject `LogActionService` and emit an entry with the
   `ACTION_TYPE` value and actor context on every create/update/delete.
3. **Named DB connection on every `@InjectRepository`** — always
   `@InjectRepository(Entity, DB_TYPE.DB_XXX)`. Never the default connection.
4. **Batch over loop** for multi-row writes.
5. **ClickHouse**: read-only, parameter-bound (`query_params` + typed placeholder), never string
   interpolation.
6. **Uploaded media** goes through `synCDN`/`synCDNImages`; persist the CDN-returned path
   (`limg_path`), not the local disk path.
7. **Static route segments declared above `:param` routes** — Nest matches in declaration order.
8. **Derived columns resolved server-side**, never trusted from the client payload.

## Reference implementation

`src/modules/config/referral/referral-campaign` already follows the service/repository/folder
parts of this ADR (plain classes, no interfaces, no base repo, referral folder shape).

Two caveats before copying it wholesale:
- Its controller still extends `BaseController` and try/catches every handler — that part is
  **superseded** by this ADR.
- It declares `@Get(':id')` above `@Get('report')`, which breaks the report route. Do not copy the
  ordering.
