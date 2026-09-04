---
title: Glossary
tags:
  - glossary
updated: '2026-09-04'
summary: >-
  Domain terms shared across repos. English definitions; Vietnamese terms of art
  kept verbatim with a gloss.
status: draft
links:
  - repos/api.md
---
# Glossary

Domain terms shared across repos. Each entry gets a short definition so every agent (and person)
uses the same word for the same concept.

**Language rule**: definitions are written in English. A Vietnamese term of art — a domain word, an
enum name, a DB value, a UI label — is kept **verbatim** as the entry name, with an English gloss
after the dash. Do not translate away an identifier that appears in the code.

## Domain terms

- **doi soat** (`thống kê`; connection `DB_DOISOAT_WRITE`) — reconciliation / settlement between
  partners.
- **TK** (`thống kê`; connection `DB_TK_WRITE`) — statistics. The abbreviation appears in
  connection names and module names, not spelled out.
- **chien dich** (`chiến dịch`) — campaign. Used for referral campaigns and promotions; the
  referral controller's Swagger tag is Vietnamese.
- **Referral campaign** (`REFERRAL_CAMPAIGN`) — a time-bounded invite programme: a referrer invites
  a referee, who activates a package. Owns banners, a package/product pair, and a reward cap.
- **Referrer / referee** (`referrer_member_id` / `referee_member_id`) — the member who sends an
  invite vs. the member who accepts it. Counted from different stores: referrers from ClickHouse
  (`log_referral_inviter`), referees/activations from MySQL (`REFERRAL_TRANSACTION`).
- **Activation** — a `REFERRAL_TRANSACTION` row with `status = 1` (SUCCESS). This is the definition
  used by campaign reporting; it is not a separate table or column.
- **Package vs product** (`package_id` / `product_id`) — `package_id` is the billing cycle supplied
  by the client; `product_id` is **derived** from it server-side via `MULTISCREEN_ECO_PACKAGE`
  (`DB_CONTENT`), defaulting to `0` when no active package matches. Never accept `product_id` from
  a client payload.
- **`limg_path`** — the CDN-returned path produced by `synCDN`. Entity banner/image columns store
  this, never the local upload path.

> Seeded from terms verified during the 2026-09-03 core-refresh of the referral module and the
> `DB_TYPE` enum. Still `draft` — the repo has many more domain terms (content, comic, loyalty,
> sporthub, b2b) that are not captured here yet.

## Frontend terms (`smartcms` web)

Added 2026-09-04 when the frontend repo was documented — see [`repos/web.md`](repos/web.md).

- **SmartCMS** — the project's own name, used for both sides (`smart-cms` in the GitLab group,
  `smartcms` in the pm2 process names). "CMS web panel" and "SmartCMS Admin" refer to the same
  frontend app.
- **elstar** — the admin dashboard template the frontend was forked from. It is still the
  `package.json` `name` and shows up in template component paths. It is **not** the project name
  and carries no meaning; do not treat `elstar` as a domain concept.
- **`CommonDataResponse2<T>`** (`src/@types/common.ts`) — the frontend's type for the shared
  response envelope. The `2` is a version suffix on the type, not on the API.
- **`injectReducer`** — the frontend's lazy reducer registration. Each view calls it once on mount;
  a feature's reducer is never added to the global `rootReducer.ts`.
- **`NormalDataTable` / `SelectableDataTable`** — the two shared data-table wrappers in
  `@/components/custom/DataTable`. Kendo-backed, and the one place new frontend code still lands on
  the legacy UI kit because Mantine has no table. `Selectable` is only for checkbox-driven bulk
  actions.
- **ActionBar** — the per-module header/filter/bulk-action bar that sits above a list table. A
  naming convention, not a shared component: each module has its own.
