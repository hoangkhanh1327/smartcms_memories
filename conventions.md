---
title: Conventions
tags:
  - conventions
updated: '2026-09-04'
summary: >-
  Shared conventions, split into cross-repo (API + frontend), API-only and
  frontend-only parts. All four parts are now written.
status: ready
links:
  - architecture/decisions/0001-new-feature-module-conventions.md
  - architecture/decisions/0002-frontend-mantine-design-system.md
  - architecture/decisions/0003-frontend-shared-tables-tanstack.md
  - repos/api.md
  - repos/web.md
---
# Conventions

This file has **four active parts**. Do not apply the API-only part to frontend code, or the
frontend-only part to backend code.

- **Part 1 — Cross-repo**: binds both the API and the CMS web panel. Changing anything here is a
  breaking change for the other side.
- **Part 2 — API-only** (`api-smart-cms`): backend/NestJS specifics.
- **Part 3 — Frontend-only** (`smartcms` web): React/Vite specifics.
- **Part 4 — Agent working rules**: how agents must use and update this memory.

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

On the frontend this envelope is typed `CommonDataResponse2<T>` in `src/@types/common.ts`. Note how
it is unwrapped — see Part 3 and `repos/web.md` § API layer.

## Route prefix

All API routes live under the prefix from the backend's `BASE_URL` env var, deployed as `api/v1`
→ `https://<host>/api/v1/<path>`. The frontend reaches it through `VITE_API_URL` →
`appConfig.apiUrl` → `BaseService`'s `baseURL`; several *other* backends have their own
`VITE_API_*_URL` and their own service module.

## Bulk delete

Bulk deletes are `POST <resource>/delete` with an `ids` body — **not** `DELETE <resource>/:id`.
`ids` accepts either a real array or a comma-separated string (`"1,2,3"`).

## Status enum convention

Entity `status` columns are `tinyint`, `0` = INACTIVE / `1` = ACTIVE. Transaction-style statuses
differ per table (e.g. referral transactions use `0` PROCESSING / `1` SUCCESS / `2` FAILED) — check
the specific table rather than assuming.

**`0` is a real value, not "empty".** Because ACTIVE/INACTIVE is `1`/`0`, any frontend code that
guards a status with `||` (`value || ''`, `status || undefined`) silently turns INACTIVE into "no
value" — a select blanks itself, a filter drops out of the query. Use `??` for status, ids and any
other numeric domain value. Same trap server-side in a `if (!status)` guard.

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

# Part 3 — Frontend-only conventions (`smartcms` web panel)

Applies to `/home/khanhth/projects/admin2/web`. See [`repos/web.md`](repos/web.md) for repo shape,
[ADR 0002](architecture/decisions/0002-frontend-mantine-design-system.md) for the design-system
decision and [ADR 0003](architecture/decisions/0003-frontend-shared-tables-tanstack.md) for data
tables. These rules are ratified in the repo's own (Vietnamese) `.ai/core/constitution.md` and
expanded in `.ai/core/project-rules.md`.

## Architectural rules

1. **The service layer is the only API boundary.** No component, view, store slice or utility calls
   axios or `fetch` directly — everything goes through a function in the module's `services/`
   folder that uses `ApiService`.
2. **Views are thin.** A `views/*.tsx` may only: `injectReducer`, set `document.title`, and return a
   `<Container>` wrapping the list/form component plus its dialog provider. No fetching, no
   business logic.
3. **UI state in the slice, server state in RTK Query.** Dialog open/close, visible columns and
   filters live in the Redux slice; paginated lists and detail records live in the RTK Query
   service. **Row selection is the exception** — it belongs to the container (rule 8).
4. **No business logic in components.** Validation → Yup schema co-located with the form; derived
   data → `utils/`; workflow decisions → the RTK Query mutation hooks.
5. **Every form input is validated before submission.** A Yup schema is mandatory. New forms bind it
   with `@mantine/form` + `mantine-form-yup-resolver`; Formik + Yup is legacy-only.
6. **Reducers are injected lazily per view.** Never register a feature reducer in the global
   `rootReducer.ts`.
7. **List pages compose as Container + ActionBar + a shared table.** The container owns query
   params, fetching, pagination and selected ids. Tables come from
   **`@/components/shared/tables`** — `NormalTable` is the default, `SelectableTable` **only** when
   checkbox selection drives bulk actions. The Kendo-backed `custom/DataTable/*DataTable` wrappers
   are **frozen**: still imported by not-yet-migrated screens, never edited, never chosen for new
   work (ADR 0003).
8. **Selection state has exactly one owner** — the container. Pass ids down to both the table and
   the action bar; do not duplicate them in `ActionBar`, in the table wrapper, or in the slice.
9. **No ad-hoc HTML/CSS control styles.** New controls come from `@mantine/core`, falling back to
   the shared wrappers only where Mantine has no equivalent wired up.

## Component placement — the decision order

1. **Mantine first.** Reach for `@mantine/core` before any legacy bucket (`@/components/ui`, Kendo,
   `twin.macro`). For the actual API, read `node_modules/@mantine/*/lib/**/*.d.ts` — **not**
   `mantine_llm.txt`, which is only a link index (ADR 0002 § Trap). The one widget class Mantine
   does not cover is the **data table**: that comes from `@/components/shared/tables` (ADR 0003),
   never from Kendo directly.
2. **One consumer → the component stays in its module** (`src/modules/<domain>/<Feature>/components/`).
   Do not pre-emptively promote.
3. **Two or more consumers → it MUST move to `src/components/shared`.** Hard rule, not a judgment
   call: promote it in the same change. Two symptoms mean the rule is *already* broken and a
   promotion is owed — a module importing from another module's `components/`, or two
   near-identical components differing only in data source or labels.
4. **`shared/` is organised by category folder, like a component library — never a flat dump.**
   Categories, each with its own barrel: `inputs/`, `feedback/`, `layout/`, `data-display/`,
   `actions/`, `utils/`, `tables/`. `src/components/shared/index.ts` re-exports every category
   barrel, so `@/components/shared` stays the single import path — always import from there, never
   deep into `shared/<category>/<File>`. (`tables/` is the exception that proves the rule's point:
   it is imported as `@/components/shared/tables`, its own barrel, because it is a subsystem rather
   than a component.) *Migration is incremental*: `shared/` is still 26 loose files; new components
   go into a category immediately, existing ones move when next touched. Never attempt it as one
   sweeping change.
5. **`ui/`, `custom/` and `common/` are legacy buckets** — extend only to fix an existing component
   in place, and `custom/DataTable` not even for that (ADR 0003).

## Naming

| Thing | Convention |
|---|---|
| View files | kebab-case (`campaign-list.tsx`) |
| Component files/folders | PascalCase (`ReferralCampaignForm/`) |
| Service functions | `api` + verb + entity, camelCase (`apiGetGamiCampain`) |
| Redux slice | `<module>Slice.ts`, name constant `<MODULE>_SLICE_NAME` |
| RTK Query service | `<module>Queries.ts` · Reducer: `<module>Reducer.ts` |
| Types | PascalCase; requests end `Request`, responses end `Response` |
| Entity field constants | SCREAMING_SNAKE_CASE (`MENU_ID`) — mirrors the API's field naming |

A secondary view inside a module (e.g. a report screen) gets its **own** slice+reducer pair
(`referralCampaignReportSlice.ts`), not a fold-in to the primary one.

Imports use the `@/` alias for anything cross-module; module-local types are imported relatively.

## Design tokens

`docs/design-system.MD` (Tailwind: colors as `themeColor` + `primaryColorLevel`, the `CONTROL_SIZES`
scale, breakpoints) is the **source of truth**, and `src/configs/mantine.theme.ts` derives Mantine's
theme from it at build time. Do not hardcode a color or a control height, and do not let Mantine
fall back to its stock theme.

**The UI language is Vietnamese** — page titles, labels, validation messages and Swagger-facing
strings are Vietnamese even though identifiers are English.

---

# Part 4 — Agent working rules

## Write this memory in English

**All memory content is English.** Every file in this repo — body prose, headings, and the
frontmatter `title` / `summary` fields — is written in English, regardless of the language the
developer used in the conversation that produced it. English is materially cheaper in tokens than
Vietnamese for the same content, and this memory is re-read at the start of every session by every
agent, so the cost is paid repeatedly.

The single exception is a **term of art that must be quoted exactly**: a Vietnamese domain word, a
DB value, an enum name, a UI label, or a `reason` a human wrote. Keep those verbatim and add an
English gloss:

```
- **doi soat** (`DB_DOISOAT_WRITE`) — reconciliation/settlement between partners.
```

This applies to `memory_write` content and to the `reason` argument. It does **not** ask anyone to
change how they talk: converse with the developer in whatever language they use, then write the
memory in English.

If you find an existing memory file written in another language, rewrite it into English the next
time you touch that file — do not leave a mixed-language file behind.

## Update this memory at commit time

Applies to every agent tool (Claude Code, Copilot, OpenCode, Antigravity). Mirrored in the repo's
`AGENTS.md`, and enforced for Claude Code by a `PreToolUse` hook on `git commit`
(`.claude/hooks/memory-commit-reminder.sh`) that fires when new module files are staged.

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
| Change to the response envelope, route prefix, or any API↔frontend contract | `conventions.md` **Part 1** — flag it as breaking for the other side |

**Do not write**: conversation logs, per-task progress, temporary notes, or bug fixes with no
architectural consequence. This is a decision record, not a changelog — `git log` covers what
changed. Typo fixes, lint passes, dependency bumps and concept-free refactors need no entry. When
unsure, ask the developer instead of writing noise.

**Mechanics**:
- `reason` becomes the memory repo's commit message — write a real one, in English.
- Prefer `mode: "append"`; `overwrite` only when existing content is genuinely obsolete.
- `status: "ready"` only once a decision is settled; leave `draft` while still under discussion.
- `links` are validated on write — create the linked file first, then the file that links to it.
- Respect scope: an ADR under `architecture/decisions/` states its own scope in its Scope section.
  ADR 0001 is **API-only**; ADR 0002 and ADR 0003 are **frontend-only**. Never put a rule from one
  side where the other will read it as shared.
