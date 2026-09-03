---
title: Conventions
tags:
  - conventions
updated: '2026-09-03'
summary: 'Quy ước naming, format response, error codes dùng chung.'
status: ready
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
---
# Conventions

> Quy ước chung: naming, format response, error codes... áp dụng cho mọi repo.

## Naming

- (quy ước đặt tên biến/hàm/route chung nếu có)

## Error codes

- (mã lỗi chuẩn dùng chung giữa các service, nếu có)

## New feature modules — do NOT copy the legacy patterns

Building a **new** feature module in `api-smart-cms`? Read
[`architecture/decisions/0001-new-feature-module-conventions.md`](architecture/decisions/0001-new-feature-module-conventions.md)
**before** writing code. Short version:

- **No** `BaseRepository` / base-service class — plain `@Injectable()` classes.
- **No** `IXxxRepository` / `IXxxService` interfaces and **no** string DI tokens — inject the
  concrete class.
- **No** `BaseController` — return plain values; the envelope comes from a per-module interceptor
  that must emit `{ status, result, message, data, errors }` (note `errors`, plural).
- Throw `HttpException` subclasses, never plain `new Error(...)` — a plain `Error` leaks the stack
  trace to the client via `AllExceptionFilter`.
- Folder shape: thin parent aggregator module + leaf folders; only `dto/` and `entities/` are
  subdirectories, everything else flat at the leaf root.

Applies to **new modules only**. Existing modules keep their current style — do not migrate them.
`.ai/core/project-rules.md` describes the *legacy* conventions (`provenance: inferred`); where the
two disagree for new code, this ADR wins.

## Response envelope (all endpoints, old and new)

```jsonc
{ "status": 200, "result": 0, "message": "success", "data": {}, "error": null }
```

`result` is `0` on success and `-1` on failure. Note the wire field is `error` (singular) while the
internal `BaseResponse` property is `errors` (plural); the global `TransformInterceptor` does the
rename.
