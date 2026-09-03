---
title: Conventions
tags:
  - conventions
updated: '2026-09-03'
summary: >-
  Quy ước dùng chung. Chia 2 phần: cross-repo (API + frontend) và API-only.
  Frontend-side sẽ thêm sau.
status: ready
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
  - repos/api.md
---
# Conventions

This file has **two parts**. Do not apply the API-only part to frontend code.

- **Part 1 — Cross-repo**: binds both the API and the CMS web panel. Changing anything here is a
  breaking change for the other side.
- **Part 2 — API-only** (`api-smart-cms`): backend/NestJS specifics.
- **Part 3 — Frontend-only**: not written yet; owner will add it in a later pass.

---

# Part 1 — Cross-repo conventions

## HTTP response envelope

Every endpoint, legacy and new, returns this shape:

```jsonc
{ "status": 200, "result": 0, "message": "success", "data": {}, "error": null }
```

- `result` is `0` on success, `-1` on failure.
- `status` mirrors the HTTP status code.
- The wire field is **`error`** (singular). Internally the backend's `BaseResponse` property is
  `errors` (plural) and the global `TransformInterceptor` does the rename — a backend detail, but
  it explains why both spellings appear in API code.

Frontend clients should read `result` / `status` for success detection, not the presence of `data`
(a successful empty response still carries `data`).

## Route prefix

All API routes live under the prefix from the backend's `BASE_URL` env var, deployed as `api/v1`
→ `https://<host>/api/v1/<path>`.

## Bulk delete

Bulk deletes are `POST <resource>/delete` with an `ids` body — **not** `DELETE <resource>/:id`.
`ids` accepts either a real array or a comma-separated string (`"1,2,3"`).

## Status enum convention

Entity `status` columns are `tinyint`, `0` = INACTIVE / `1` = ACTIVE. Transaction-style statuses
differ per table (e.g. referral transactions use `0` PROCESSING / `1` SUCCESS / `2` FAILED) — check
the specific table rather than assuming.

---

# Part 2 — API-only conventions (`api-smart-cms`)

Applies to `/home/khanhth/projects/admin2/api`. See [`repos/api.md`](repos/api.md) for repo shape.

## New feature modules — do NOT copy the legacy patterns

Building a **new** feature module? Read
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
`.ai/core/project-rules.md` in the repo describes the *legacy* conventions (`provenance: inferred`)
and is tagged `[legacy-only]` where ADR 0001 overrides it.

## Cross-cutting rules that apply to all backend code

1. Controller → Service → Repository. No DB access from a controller. ADR 0001 removes the base
   classes and interfaces, **not** the layers.
2. Audit log on every mutation via `LogActionService` + the `ACTION_TYPE` enum.
3. Always name the DB connection: `@InjectRepository(Entity, DB_TYPE.DB_XXX)`. Never the default.
4. Batch over loop for multi-row writes.
5. ClickHouse is read-only and parameter-bound (`query_params` + typed placeholder). Never string
   interpolation.
6. Uploaded media goes through `synCDN`/`synCDNImages`; persist the CDN-returned path
   (`limg_path`), not the local disk path.
7. Static route segments declared **above** `:param` routes — Nest matches in declaration order.
8. Derived columns resolved server-side, never trusted from the client payload.

---

# Part 3 — Frontend-only conventions (CMS web panel)

*Not documented yet.* Owner will add this in a later pass. When adding it, keep frontend rules in
this section (or their own ADR with an explicit scope note) and only promote a rule into Part 1 if
it genuinely binds both sides.

---

# Part 4 — Agent working rules

## Update this memory at commit time

Applies to every agent tool (Claude Code, Copilot, OpenCode, Antigravity). Mirrored in the repo's
`AGENTS.md`.

**Session start**: call `memory_list` before assuming any convention. `OVERVIEW.md` is the entry
point, and it tells you which repo you're in.

**Before committing**: if the change introduces any of the below, write it here **in the same
commit cycle**, not later.

| Commit introduces | Write to |
|---|---|
| New feature or module | `repos/<repo>.md`, plus `contracts/` if an endpoint contract changed |
| New architecture decision, or a deliberate break from an existing convention | new `architecture/decisions/NNNN-<slug>.md` |
| New convention other code must follow | `conventions.md`, in the correct Part |
| New domain term, or a common word with project-specific meaning | `glossary.md` |
| Change to the response envelope, route prefix, or any API↔frontend contract | `conventions.md` **Part 1** — flag it as breaking for the frontend |

**Do not write**: conversation logs, per-task progress, temporary notes, or bug fixes with no
architectural consequence. This is a decision record, not a changelog — `git log` covers what
changed. Typo fixes, lint passes, dependency bumps and concept-free refactors need no entry. When
unsure, ask the developer instead of writing noise.

**Mechanics**:
- `reason` becomes the memory repo's commit message — write a real one.
- Prefer `mode: "append"`; `overwrite` only when existing content is genuinely obsolete.
- `status: "ready"` only once a decision is settled; leave `draft` while still under discussion.
- `links` are validated on write — create the linked file first, then the file that links to it.
- Respect scope: `architecture/decisions/` is API-side unless the ADR says otherwise. Never put a
  backend-only rule where the frontend will read it as shared.
