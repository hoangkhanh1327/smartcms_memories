---
title: SmartCMS Project — Overview
tags:
  - overview
updated: '2026-09-04'
summary: >-
  Entry point for the memory repo. The SmartCMS project has an API side and a
  frontend side; both are now documented.
status: ready
links:
  - conventions.md
  - repos/api.md
  - repos/web.md
---
# SmartCMS Project — Overview

> Entry point for this memory repo. Agents should read this file first in every session.

**Memory language: English only.** See `conventions.md` Part 4 before writing anything here.

This memory repo covers the **SmartCMS project** — the back-office CMS for a VOD/media platform
(movies, music clips, comics, content bundles, plus platform display configuration). It spans more
than one codebase; read the repo map below before assuming a rule applies to the code you're
working on.

## Repos

| Repo | Side | Memory |
|---|---|---|
| `api-smart-cms` (`/home/khanhth/projects/admin2/api`) | **Backend API** — NestJS 10 + Fastify, MySQL/MongoDB/Redis/ClickHouse | [`repos/api.md`](repos/api.md) |
| `smartcms` web (`/home/khanhth/projects/admin2/web`, `gitlab.mytv.vn:cms/smart-cms/web`) | **Frontend** — React 18 + Vite SPA, Redux Toolkit + RTK Query, Mantine 7 over a Kendo React legacy | [`repos/web.md`](repos/web.md) |

## How to use this memory

- **Every ADR under `architecture/decisions/` declares its own scope in its Scope section — read it
  before applying the ADR.** ADR 0001 is **API-only** (NestJS module conventions); ADR 0002 is
  **frontend-only** (Mantine 7 design system). Never cross them.
- `conventions.md` has four parts: **Part 1 cross-repo** (things both sides must agree on, e.g. the
  HTTP response envelope), **Part 2 API-only**, **Part 3 frontend-only**, **Part 4 agent working
  rules**. Respect the split; only promote a rule into Part 1 if it genuinely binds both sides.
- Both repos run **ai-framework** internally (`AGENTS.md` + `.ai/core/*.md` + `.ai/decisions/`).
  Those in-repo docs are `provenance: inferred` and often describe *legacy* patterns — this memory
  records what is actually decided. Where they disagree, this memory wins; note the disagreement
  rather than silently following the repo doc.

## Data flow

```
smartcms web (React 18 + Vite SPA)
        │  HTTP/JSON via axios, prefix from BASE_URL (deployed as api/v1)
        │  JWT Bearer from the redux-persist blob; 401 → signOutSuccess()
        ▼
api-smart-cms (NestJS 10 + Fastify)
        └─ MySQL (~19 named connections) / MongoDB / Redis / ClickHouse (read-only) / CDN sync
```

The **response envelope is the contract between the two sides** — see `conventions.md` Part 1. Any
change to it is a breaking change for the frontend. Note that the frontend's axios interceptor
unwraps one level, so a service call site's `res.data` is already the payload — the detail is in
`repos/web.md` § API layer, and it is the single easiest thing to get wrong when touching the
frontend's service or RTK Query code.

The frontend also talks to **more than one backend host** (`VITE_API_URL`, `VITE_API_GATEWAY_URL`,
`VITE_API_LOG_REALTIME_URL`, `VITE_API_URL_SHORT`, `VITE_API_WEBAPP_URL`); `api-smart-cms` is only
the main one.
