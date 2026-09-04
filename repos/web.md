---
title: smartcms web — Frontend repo
tags:
  - repo
  - frontend
  - react
  - mantine
updated: '2026-09-04'
summary: >-
  SmartCMS Admin — the CMS web panel. React 18 + Vite SPA, Redux Toolkit + RTK
  Query, Mantine 7 for new UI over a large Kendo React legacy. Consumes
  api-smart-cms. Local machine cannot run a full build/typecheck.
status: ready
links:
  - architecture/decisions/0002-frontend-mantine-design-system.md
  - conventions.md
  - architecture/decisions/0003-frontend-shared-tables-tanstack.md
---
# `smartcms` web — Frontend

**Path**: `/home/khanhth/projects/admin2/web` · **Git**: `git@gitlab.mytv.vn:cms/smart-cms/web.git`
**Role**: SmartCMS Admin — the back-office web panel that consumes `api-smart-cms`.

Pure client-side SPA; there is **no server code in this repo**. The `package.json` `name` is still
`elstar` (the admin template it was forked from) — the project is *smartcms*, not elstar.

Audience: internal admins and editors managing VOD/media content (movies, music clips, comics,
content segments, bundles) plus system-level display configuration. **UI language is Vietnamese**
(a constitution Non-Negotiable) — page titles, labels and validation messages are written in
Vietnamese even though code identifiers are English.

## Stack

- **React 18.2** + **TypeScript 4.9.5** (old — see the trap below), built by **Vite 4**.
- **Redux Toolkit 1.9** + `react-redux` 8 + `redux-persist` — slices for UI state, **RTK Query**
  for server state. Reducers are injected **lazily per view** (`injectReducer`), never registered
  in the global `rootReducer.ts`.
- **react-router-dom 6.30**, routes declared in a config tree (see below), views `lazy()`-loaded.
- **axios 1.3** + `axios-retry` behind a service layer. No component calls `fetch`/axios directly.
- **UI**: **Mantine 7.17** for all new work; **Kendo React 4.x** (399 files) and `twin.macro` are
  maintain-only legacy — see
  [ADR 0002](../architecture/decisions/0002-frontend-mantine-design-system.md).
- **TailwindCSS 3.3** — not just styling: it is the **token source** (colors, control sizes,
  breakpoints) documented in `docs/design-system.MD`, and Mantine's theme is derived from it.
- Forms: `@mantine/form` + `mantine-form-yup-resolver` (new) / **Formik** + Yup (legacy).
- Also present: Bitmovin player + hls.js/video.js, ApexCharts + Recharts, `@dnd-kit`,
  `@tanstack/react-table` + `react-virtual`, exceljs/xlsx export, TinyMCE + Quill + TipTap.

## Repo layout

```
src/
  modules/<domain>/<Feature>/   # feature code — 16 domains: content, configs, administration,
                                # auth, bundle, businessRule, HST, log, partnerManagement,
                                # player, recommendation, search, statistics, common, …
  services/                     # global API layer — BaseService, ApiService, RtkQueryService, …
  store/                        # root store, rootReducer, injectReducer, auth/theme/base slices
  components/{shared,ui,custom,common,layouts,route,template,player}
  configs/                      # app.config, mantine.theme, routes.config/, navigation.config/
  @types/  constants/  utils/  assets/
```

**Per-module internal layout** (uniform across modules):

```
<Feature>/
  @types/      # <entity>.types.ts + <entity>Services.types.ts
  services/    # api* functions — always through ApiService, never axios
  stores/      # <module>Slice.ts + <module>Queries.ts + <module>Reducer.ts
  views/       # route-level pages — thin: injectReducer, document.title, render
  providers/   # dialog-state context providers
  components/{lists,forms,dialogs,selects,editors,uploads}
  utils/  constants/  docs/
```

**Routes live outside the module**, in a parallel config tree:
`src/configs/routes.config/configs/<module>/<module>Route.tsx`, each default-exporting a `Routes`
array of `{ key, path, component: lazy(...), meta }`. `meta.pageContainerType: 'gutterless'` for
full-width list/report screens; `'default'` + `meta.header` (Vietnamese title) for create/edit forms.

**Reference module**: `src/modules/configs/referral` — the newest and the one to copy for structure,
Mantine usage, and RTK Query typing.

## API layer — the one contract to get right

`src/services/BaseService.ts` installs `interceptors.response.use((response) => response.data)`.
**Every axios call therefore resolves to the HTTP body, not an `AxiosResponse`.** `ApiService.ts`
then does `resolve(response as Response)` — which *looks* like an unwrapping bug and is not.

So for `ApiService.get<GetXResponse>(url)` where `GetXResponse = CommonDataResponse2<T>`
(`src/@types/common.ts`):

- the resolved value **is** the envelope `{ status, result, message, data }`
- and `res.data` **is** `T` — the real payload

That is why every service call site reads `res.data` and gets the entity directly. Do not "fix" one
by adding another `.data`. Consequence for RTK Query: the endpoint's result generic is the **inner**
payload — `build.mutation<ReferralCampaign, CreateReferralCampaignRequest>` for a service returning
`CommonDataResponse2<ReferralCampaign>`, with `queryFn` returning `{ data: res.data }`. Reference:
`stores/referralCampaignQueries.ts`.

`ApiService.fetchData` is the exception — it is typed `Promise<AxiosResponse<Response>>`, which the
interceptor makes untrue. Prefer the verb helpers (`get`/`post`/`patchFormData`/…).

Other API-layer facts:
- Auth: JWT read from the `redux-persist` blob in `localStorage` and sent as a Bearer header; a
  `401` response dispatches `signOutSuccess()` from the interceptor.
- Base URLs come from `import.meta.env.VITE_*` via `src/configs/app.config.ts` — several distinct
  backends (`apiUrl`, `apiGatewayUrl`, `apiRealtimeUrl`, `apiUrlShort`, `apiWebappUrl`) plus three
  CDN hosts. There is more than one API host; check which service module you are in.
- Per-endpoint timeout overrides live in `src/services/customTimeoutConfig.ts` (default 120 s).

## Build, deploy, CI

- Dev: `yarn start` (Vite dev server). Build: `yarn build:{stag,stag2,pilot,prod}` — mode-specific
  `.env.*` files. Output goes to `build/`, served by `pm2 serve ./build <port> --spa`.
- GitLab CI (`.gitlab-ci.yml`) has a single `deploy` stage, one job per branch
  (`stage`, `stage_v2`, `hotfix/*`), each running a script in `bash/`. **No test or lint job runs
  in CI** — there is no test suite in this repo.

## Traps on this machine — read before running anything

1. **A full `tsc --noEmit` or `vite build` OOMs here.** The local machine lacks the RAM, even at
   `--max-old-space-size=6144`. Do not reach for `yarn build` as a verification step.
   **Working recipe** — a *module-scoped* tsc run succeeds at only 2048 MB and is fast enough to use
   as a real regression net:

   ```bash
   # scoped.json extends the repo tsconfig, overriding include[] to just the paths you touched
   node --max-old-space-size=2048 node_modules/typescript/lib/tsc.js -p /path/scoped.json \
     2>&1 | grep -v '^node_modules/'
   ```

   Add `--noUnusedLocals --noUnusedParameters` (neither is on in the repo config) to catch dead
   imports. The `grep -v` matters: `@mantine/core`'s `.d.ts` files always produce parse errors under
   TypeScript 4.9.5 and are not yours.
2. **That parse failure means Mantine props degrade to `any`** — a scoped tsc run passes even with
   an invented prop. It verifies *your* code's types, never Mantine prop correctness. Check those
   against `node_modules/@mantine/*/lib/**/*.d.ts`, and **do not** use `mantine_llm.txt`: despite
   what the repo's own rules and the prompt hook say, it is only a list of doc URLs. See ADR 0002.
3. **`npx eslint` fails immediately** — `.eslintrc.cjs` references `eslint-plugin-prettier`, which
   is not installed.
4. The Vite **dev server** runs fine (~1 s startup): `npx vite --port <n> --strictPort`. A headless
   Chromium exists at `~/.cache/ms-playwright/chromium-1234/chrome-linux64/chrome` (no playwright
   package installed — drive it over CDP with Node 22's native `WebSocket`) if a change really needs
   visual confirmation.
5. `jq` is not installed on this machine (same as the API repo). Use `node` or `/usr/bin/python3`.

## In-repo agent framework

This repo runs **ai-framework**: `AGENTS.md` is the canonical entry point, `.ai/core/*.md` holds the
derived docs (`constitution.md` is ratified and **written in Vietnamese**, v0.3.0), `.ai/decisions/`
holds in-repo ADRs, `.ai/features/NNN-*/` holds the per-feature
brainstorm→clarify→plan→tasks lifecycle. Claude Code skills live in `.claude/skills/`.
A `.codegraph/` index exists — `codegraph query -p <repo> <term>` beats grepping blind.

Frontend coding conventions are in `conventions.md` Part 3; the design-system decision is
[ADR 0002](../architecture/decisions/0002-frontend-mantine-design-system.md).

## Data tables — `shared/tables`, and `custom/DataTable` is frozen

Added 2026-09-04, see [ADR 0003](../architecture/decisions/0003-frontend-shared-tables-tanstack.md).

`src/components/shared/tables` is a **TanStack Table v8 + TanStack Virtual** rebuild of the Kendo
Grid wrappers, exporting `NormalTable`, `SelectableTable`, `EditableTable`, `SortTable`, `SpanTable`,
`TreeTable` plus `DataTableCore` and the `CommonTableColumnProps` / `TableCellProps` /
`TablePageChangeEvent` type family from one barrel: `@/components/shared/tables`. Its `README.md`
holds the authoritative old→new symbol mapping and the list of deliberate behaviour changes.

**`src/components/custom/DataTable` is frozen** — not edited, not extended, not chosen for new work;
it only serves screens not yet migrated. ~870 files still import it and ~3,095 `renderCell`
callbacks return raw `<td>` elements, which is exactly why the contract was kept identical: a screen
migrates by changing imports, and a `renderCell` returning a `<td>` still works (the element is
adopted as the cell, with frozen-column class/offsets merged in). Kendo packages and
`@progress/kendo-theme-default/dist/all.css` (imported at `src/App.tsx`) stay until the last screen
moves.

So the "Also present: `@tanstack/react-table` + `react-virtual`" line in the Stack section above is
no longer incidental — it is the table stack. `src/modules/configs/referral` already imports from
`@/components/shared/tables`; copy it rather than an older list screen.
