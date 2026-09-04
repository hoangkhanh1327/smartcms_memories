---
title: ADR 0002 — Mantine 7 is the design system for new frontend UI
tags:
  - conventions
  - architecture
  - frontend
  - react
  - agent-rule
updated: '2026-09-04'
summary: >-
  FRONTEND-SCOPED. New UI in the smartcms web repo is built on Mantine 7; Kendo
  React and twin.macro are maintain-only legacy; Mantine's theme is derived from
  the repo's Tailwind tokens. Does not apply to the API repo. Amended by ADR
  0003 — data tables now come from shared/tables, not custom/DataTable.
status: ready
links:
  - conventions.md
  - architecture/decisions/0003-frontend-shared-tables-tanstack.md
---
# ADR 0002 — Mantine 7 is the design system for new frontend UI

**Status**: accepted, **amended 2026-09-04 by**
[ADR 0003](0003-frontend-shared-tables-tanstack.md) (data tables) · **Date**: 2026-09-03 ·
**Decided by**: project owner

## Scope — read this first

**This ADR is FRONTEND-ONLY.** It governs the `smartcms` web repo
(`/home/khanhth/projects/admin2/web`) and nothing in `api-smart-cms`. It is the counterpart to
ADR 0001, which is API-side only — the two never apply to the same code.

It mirrors, in this memory repo, the in-repo record at `.ai/decisions/0001-mantine-design-system.md`
and the `constitution.md` v0.3.0 amendment (the repo's own constitution is written in Vietnamese;
this file is its English record for cross-repo agents).

## Context

`src/` carried **three overlapping UI kits with no rule choosing between them**:

| Kit | Files at decision time | Role |
|---|---:|---|
| `@progress/kendo-react-*` | 399 | the incumbent |
| `@mantine/*` 7.17.0 | 22 | the new direction |
| `twin.macro` | 4 | near-vestigial |

Mantine was not spread evenly — 13 of 22 files sat in `src/modules/configs/referral`, the newest
module. Mantine had already become the de-facto choice for new work, but nothing recorded it, so
every feature re-litigated the question.

Complicating it: `docs/design-system.MD` documents the **Tailwind** token system (colors as a
`themeColor` + `primaryColorLevel` pair, a control-size scale, breakpoints) and never mentions
Mantine, while `<MantineProvider>` in `src/App.tsx` was mounted with no `theme` prop — Mantine ran
on stock defaults, visually disconnected from those tokens.

## Decision

1. **Mantine 7 is the design system for new UI.** New screens and components are built on
   `@mantine/core` and its companion packages (`form`, `dates`, `dropzone`, `notifications`,
   `modals`, `charts`, `tiptap`, `hooks`, …).
2. **Kendo React and `twin.macro` are legacy, maintain-only.** They keep working and may be
   bug-fixed where already used, but are **not** extended into new screens or components. The same
   applies to the pre-Mantine primitives in `@/components/ui` and the Kendo-era composites in
   `@/components/custom`.
3. **The Tailwind token system in `docs/design-system.MD` remains the source of truth** for colors,
   spacing and responsive behaviour. Mantine's theme is configured to follow it, not to diverge on
   defaults.
4. **Forms**: new forms use `@mantine/form` bound to the existing Yup schemas through
   `mantine-form-yup-resolver`. Formik stays in legacy modules only.

## Consequences

**Point 3 is implemented.** `src/configs/mantine.theme.ts` (`buildMantineTheme()`) reads the
palette, font stack and breakpoints through `twin.macro`'s build-time `theme` macro — no duplicated
copy that can drift. It maps `themeColor`→`primaryColor`, `primaryColorLevel`→`primaryShade`
(as a tuple index, because the repo's `ColorLevel` includes `50`), Tailwind px screens→Mantine
**em** breakpoints, and `CONTROL_SIZES` onto Mantine's `size` scale. Consequence worth knowing: a
Mantine `size="xs"` control renders at the same height as the legacy `@/components/ui` control
next to it, so mixing the two in one toolbar does not misalign.

**The split is by widget class, not by module.** `configs/referral` — the reference module — mixes
kits deliberately:

| Concern | New code uses |
|---|---|
| Form state + validation | `@mantine/form` + `mantine-form-yup-resolver` |
| Inputs, layout, buttons | `@mantine/core` |
| Date inputs / file upload / toasts / rich text | `@mantine/dates` / `@mantine/dropzone` / `@mantine/notifications` / `@mantine/tiptap` |
| **Data tables** | **`@/components/shared/tables`** (TanStack) — see the amendment below |
| Page shell | `shared/Container`, `shared/StickyFooter`, `custom/layouts/Header` |

**Amended 2026-09-04 — the data-table exception is closed.** As originally written, this ADR said
there was no Mantine data table and that new table work therefore still landed on the Kendo-backed
`custom/DataTable/NormalDataTable`. Mantine still has no data grid, but the fallback is no longer
Kendo: [ADR 0003](0003-frontend-shared-tables-tanstack.md) makes
**`@/components/shared/tables`** (TanStack Table v8 + Virtual) the table implementation for all new
and migrated work, and **freezes `custom/DataTable`** — imported by not-yet-migrated screens, never
edited. The original advice to reach Kendo event types through the wrapper's prop types rather than
importing `@progress/*` survives in a stronger form: `shared/tables` exports plain
`TableCellProps` / `TablePageChangeEvent` / … types, so module code needs no `@progress/*` import at
all. `ReferralCampaignList.tsx` is still the pattern to copy; it now imports from `shared/tables`.

## Trap — do not look up Mantine APIs the way the repo tells you to

The repo's constitution and `.ai/core/project-rules.md` both say "check `mantine_llm.txt` at the
repo root for the component's API", and a Claude Code prompt hook repeats it on every UI prompt.
**`mantine_llm.txt` is only a link index** (43 KB of `- [Button](https://mantine.dev/llms/…)` lines).
It has no prop tables, no signatures, no examples, and cannot answer a prop question.

The real lookup path is `node_modules/@mantine/<pkg>/lib/components/<Name>/<Name>.d.ts`.

This matters more here than in a normal repo: the repo pins **TypeScript 4.9.5**, which cannot parse
Mantine 7.17's `.d.ts` output, so **Mantine props silently degrade to `any` and `tsc` accepts an
invented prop**. Reading the `.d.ts` by hand is the only verification short of a browser. After a
Mantine change, say plainly that props were checked against the `.d.ts` and that nothing was
rendered in a browser — do not imply `tsc` verified them.
