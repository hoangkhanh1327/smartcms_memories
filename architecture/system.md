---
title: System Architecture
tags:
  - architecture
updated: '2026-09-03'
summary: Overall data flow between repos. API side documented; frontend side pending.
status: draft
links:
  - repos/api.md
  - OVERVIEW.md
---
# System Architecture

Overall data flow between the repos of the CMS project.

```
CMS web panel (frontend, not yet documented)
        │  HTTP/JSON, prefix from BASE_URL (deployed as api/v1)
        │  JWT Bearer, global AuthGuard
        ▼
api-smart-cms  (NestJS 10 + Fastify)
        ├── MySQL / TypeORM — ~19 named connections (DB_TYPE enum), synchronize: false
        ├── MongoDB / Mongoose — specific content sub-domains
        ├── Redis / ioredis — 5 named instances (response cache, member, recommend, vmx, ccu)
        ├── ClickHouse — READ-ONLY analytics (behavior, promotions, referral, landing page)
        └── CDN sync — synCDN / synCDNImages; entities persist the returned limg_path
```

Notes:
- There are **no cross-database JOINs**. Reporting endpoints collect ids from the primary store,
  fan out to each other store with a batched `IN (...)` inside one `Promise.all`, and join in
  memory by that id. `ReferralCampaignService.getCampaignReport` is the reference example.
- The **response envelope** is the contract between frontend and API — see `conventions.md` Part 1.
- Detail on the API side lives in [`repos/api.md`](../repos/api.md); the repo's own
  `.ai/core/architecture.md` holds the module-by-module breakdown.

> `draft` — expand once the frontend repo is documented.
