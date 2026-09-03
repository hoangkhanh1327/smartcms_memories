---
title: CMS Project — Overview
tags:
  - overview
updated: '2026-09-03'
summary: >-
  Entry point for the memory repo. The CMS project has an API side (documented)
  and a frontend side (not yet documented).
status: ready
links:
  - conventions.md
  - repos/api.md
---
# CMS Project — Overview

> Entry point for this memory repo. Agents should read this file first in every session.

**Memory language: English only.** See `conventions.md` Part 4 before writing anything here.

This memory repo covers the **CMS project**, which is more than one codebase. Read the repo map
below before assuming a rule applies to the code you're working on.

## Repos

| Repo | Side | Status in memory |
|---|---|---|
| `api-smart-cms` (`/home/khanhth/projects/admin2/api`) | **Backend API** — NestJS 10 + Fastify, MySQL/MongoDB/Redis/ClickHouse | Documented — see `repos/api.md` |
| CMS web panel | **Frontend** — the admin UI that consumes this API | **Not documented yet.** Owner will add it in a later pass. |

## How to use this memory

- **Everything under `architecture/decisions/` is API-side unless its own scope section says
  otherwise.** ADR 0001 is NestJS-specific — do **not** apply it to frontend code.
- `conventions.md` is split into a **cross-repo** part (things both sides must agree on, e.g. the
  HTTP response envelope) and an **API-only** part. Respect the split.
- When the frontend is added: give it `repos/web.md`, keep its conventions in their own section or
  ADR, and only promote a rule into the cross-repo part if it genuinely binds both sides.

## Data flow

CMS web panel (frontend) → HTTP/JSON under the `BASE_URL` prefix (deployed as `api/v1`) →
`api-smart-cms` → MySQL (many named connections) / MongoDB / Redis / ClickHouse (read-only
analytics) / CDN sync.

The **response envelope is the contract between the two sides** — see `conventions.md`. Any change
to it is a breaking change for the frontend.
