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
  - contracts/ai-dubbing.md
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

## AI Dubbing terms

Added 2026-09-04 from `src/modules/content/ai_dubling/` — full contract in
[`contracts/ai-dubbing.md`](contracts/ai-dubbing.md).

- **AI Dubbing** (folder `ai_dubling`, queue prefix `dubbing-v2`) — the AI voice-over pipeline that
  turns one source video into extra dubbed audio tracks. The repo folder is misspelled (one `b`);
  the concept is *dubbing*. **thuyet_minh** (`thuyết minh`, the JWT `task` claim sent to BigData) is
  the Vietnamese term for the same thing — voice-over narration.
- **Video Job** (`video_job`) — one source video file being processed. Parent of everything else.
- **Language Task** (`language_task`) — one target-language audio track for a Video Job: AI
  translation, generated preview, and the operator's approve/reject decision. Today exactly one is
  created per Video Job.
- **Final Output** (`final_output`) — the muxed multi-audio video built from approved Language
  Tasks in a chosen `order`. At most **2 alive** per Video Job (alive = not `DELETED`/`FAILED`).
- **Preview** (`previewUrl`, `previewReadyAt`) — the reviewable render an operator watches before
  approving. Unapproved previews **expire after 15 days** (hourly cron → `EXPIRED`).
- **softsub / hardsub** (`subType` 1 / 2) — softsub keeps subtitles as a separate track and is the
  voice-over path; hardsub burns them into the picture. They hit different BigData endpoints
  (`/api/voiceovers` vs `/api/hardsub`) with different JWT secrets.
- **shortedSub** (`shorted_sub`, `shorten_sub: 'on'|'off'`) — ask the AI to condense subtitle text
  so the dubbed speech fits the timing. Spelled "shorted", not "shortened", in both DB and payload.
- **FTP A** — the media file store the pipeline reads the source video from and writes the final
  output back to. `metaData.path` and `outputPath` are paths on it. **NFS** is the separate
  intermediate store for extracted/translated audio (`{id}__source.wav`, `{id}__translated.wav`).
- **BigData** (`DOMAIN_BIGDATA_FOR_SOFTSUB` / `_HARDSUB`) — the VNPT AI partner service that does
  the actual translation and speech synthesis. Referred to as `ai_server` in `job_step_log.executor`.
- **Executor** (`cms` | `transcoder` | `ai_server`) — which system performed a logged pipeline step.
  `transcoder` is the separate BullMQ worker service, not this API.
- **CAS update** (`updateStatusWithCAS`) — compare-and-set: a status change that only applies if the
  row is still in the expected status. A lost race surfaces to the client as 409.
- **Outbox event** (`outbox_event`) — a durable record written in the same breath as a queue
  dispatch, so a reconciliation cron can re-enqueue work the queue never received. Internal; never
  exposed to the frontend.
