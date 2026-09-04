---
title: >-
  ADR 0003 — Data tables come from shared/tables (TanStack); custom/DataTable is
  frozen
tags:
  - conventions
  - architecture
  - frontend
  - react
  - agent-rule
updated: '2026-09-04'
summary: >-
  FRONTEND-SCOPED. New and migrated table work uses src/components/shared/tables
  (TanStack Table v8 + Virtual). src/components/custom/DataTable (Kendo Grid) is
  frozen — never edited, only imported by not-yet-migrated screens. Closes the
  "tables are the Kendo exception" clause of ADR 0002.
status: ready
links:
  - architecture/decisions/0002-frontend-mantine-design-system.md
  - conventions.md
---
# ADR 0003 — Data tables come from `shared/tables` (TanStack); `custom/DataTable` is frozen

**Status**: accepted · **Date**: 2026-09-04 · **Decided by**: project owner

## Scope — read this first

**FRONTEND-ONLY**, `smartcms` web (`/home/khanhth/projects/admin2/web`). Nothing here applies to
`api-smart-cms`. This ADR **amends ADR 0002**, which named data tables as the one widget class where
new work still landed on the Kendo-backed wrapper. That exception is closed.

## Context

ADR 0002 made Mantine 7 the design system but had to carve out data tables: Mantine ships no data
grid, so new screens kept importing `src/components/custom/DataTable/*` (Kendo Grid wrappers).

That exception could not simply be removed by rewriting call sites. At decision time roughly **870
files** imported the Kendo wrapper and about **3,095 `renderCell` callbacks returned raw `<td>`
elements** — a synchronous migration was untenable, and the Kendo `Grid` was also carrying real
defects (a loading overlay that ran `document.querySelector('.k-grid-content')` on every render and
portalled into whatever it found; footer `colSpan` leaving trailing empty cells).

## Decision

1. **`src/components/shared/tables` is the table implementation for all new and migrated work.** It
   is a **TanStack Table v8 + TanStack Virtual** rebuild, imported through the single barrel
   `@/components/shared/tables`.
2. **`src/components/custom/DataTable` is frozen.** Do not edit, extend or refactor it — not even to
   fix a bug that also exists in `shared/tables`. It stays only as the fallback for screens not yet
   migrated. Fix the bug in `shared/tables` and migrate the screen instead.
3. **The column/prop contract is deliberately unchanged**, so a screen migrates by changing its
   imports and nothing else. A `renderCell` that returns a `<td>` keeps working: the returned element
   is *adopted* as the cell, and frozen-column class/offsets are merged into it even when the
   callback ignores `props.className` / `props.style`.
4. **Migration is screen by screen, never a sweep.** Kendo packages stay in `package.json` — and
   `@progress/kendo-theme-default/dist/all.css` stays imported at `src/App.tsx` — until the last
   screen is migrated.
5. **Mantine-first (ADR 0002) still governs everything else.** Tables are not a licence to reach for
   another kit: controls inside cells and toolbars are still `@mantine/core`.

## The mapping

Old → new, per `src/components/shared/tables/README.md` (the authoritative, longer version):

| Old (Kendo) | New (`@/components/shared/tables`) | Note |
|---|---|---|
| `NormalDataTable` | `NormalTable` | the default |
| `SelectableDataTable` | `SelectableTable` | checkbox column is now frozen |
| `EditableDataTable` | `EditableTable` | always emits a `<td>` now |
| `SortDataTable` | `SortTable` | drag column is now frozen |
| `SpanDataTable` | `SpanTable` | not virtualised (rows joined by `rowSpan`) |
| `TreeDataTable` | `TreeTable` | takes `renderCell` **or** Kendo's `cell` |
| `GridCellProps` | `TableCellProps` | same `dataItem`/`field`/`dataIndex`/`className`/`style` |
| `GridHeaderCellProps` / `GridFooterCellProps` | `TableHeaderCellProps` / `TableFooterCellProps` | |
| `GridPageChangeEvent` / `GridItemChangeEvent` | `TablePageChangeEvent` / `TableItemChangeEvent` | page event still `props.page.skip` / `.take` |
| `GridPagerSettings` | `TablePagerSettings` | |
| `orderBy` from `@progress/kendo-data-query` | `orderBy` from `@/components/shared/tables` | |

`CommonTableColumnProps` / `CommonTableProps` keep their names — only the import path moves. A
typical migration:

```diff
-import NormalDataTable from '@/components/custom/DataTable/NormalDataTable';
-import { CommonTableColumnProps } from '@/components/custom/DataTable/@types/TableTypes';
+import { NormalTable } from '@/components/shared/tables';
+import type { CommonTableColumnProps } from '@/components/shared/tables';
```

Kendo behaviour preserved on purpose: frozen (`locked`) columns with cumulative offsets and
left-ordering, sticky header, grouped (`multiColumn`) headers, column resizing, footer cells,
toolbar, empty state, and the numeric pager with its `1 - 30 của 500 phần tử` summary (Vietnamese —
`của` = "of").

## Consequences

- **ADR 0002's "Data tables → still `custom/DataTable`" row is obsolete.** Read that row as pointing
  here. Its advice to reach Kendo event types *through the wrapper's own prop types* survives in a
  better form: the new types are plain React/TanStack types, so no module code needs `@progress/*`
  at all.
- Row **virtualisation turns on automatically past 60 rows** (`virtualize` /
  `virtualizeThreshold` to override), off for `SpanTable` and while `SortTable` drag is enabled.
  Anything that assumed every row was in the DOM (a manual `querySelector` into rows, a print view)
  must opt out explicitly.
- Column widths flow through CSS variables and the body is memoised, so resize-dragging no longer
  re-renders every row.
- Two table implementations coexist for a while. That is expected; the smell to report is an *edit*
  to `custom/DataTable`, not its continued presence.
- `src/modules/configs/referral` (the reference module of ADR 0002) already imports from
  `@/components/shared/tables` — copy it, not an older list screen.
