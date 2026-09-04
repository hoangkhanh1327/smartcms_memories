---
title: System Architecture
tags:
  - architecture
updated: '2026-09-04'
summary: >-
  Overall data flow between the SmartCMS repos. Both the frontend SPA and the
  API side are documented.
status: ready
links:
  - repos/api.md
  - repos/web.md
  - OVERVIEW.md
---
# System Architecture

Overall data flow between the repos of the SmartCMS project.

```
smartcms web  (React 18 + Vite 4 SPA — no server code, served as static files by pm2 --spa)
   │  Redux Toolkit slices (UI state) + RTK Query (server state), reducers injected per view
   │  services/ layer → ApiService → BaseService (axios 1.3 + axios-retry)
   │
   │  HTTP/JSON, baseURL = VITE_API_URL, prefix from BASE_URL (deployed as api/v1)
   │  JWT Bearer read from the redux-persist localStorage blob; 401 → signOutSuccess()
   │  response interceptor unwraps one level: res IS the envelope, res.data IS the payload
   ▼
api-smart-cms  (NestJS 10 + Fastify)
   ├── MySQL / TypeORM — ~19 named connections (DB_TYPE enum), synchronize: false
   ├── MongoDB / Mongoose — specific content sub-domains
   ├── Redis / ioredis — 5 named instances (response cache, member, recommend, vmx, ccu)
   ├── ClickHouse — READ-ONLY analytics (behavior, promotions, referral, landing page)
   └── CDN sync — synCDN / synCDNImages; entities persist the returned limg_path
```

The frontend additionally calls **other backends** directly, each with its own service module and
`VITE_API_*` host: an API gateway (`ApiGatewayService`/`GatewayService`), a realtime log service,
a URL-shortener (`ApiShortService`), a micro-service host (`ApiMicroService`), and a webapp API —
plus three CDN hosts for media (`VITE_CDN_URL`, `VITE_CDN_B2C_URL`, `VITE_CDN_STATIC`).
`api-smart-cms` is the main backend, not the only one.

## Notes

- **No cross-database JOINs on the API side.** Reporting endpoints collect ids from the primary
  store, fan out to each other store with a batched `IN (...)` inside one `Promise.all`, and join in
  memory by that id. `ReferralCampaignService.getCampaignReport` is the reference example.
- The **response envelope** is the contract between frontend and API — see `conventions.md` Part 1.
- **The frontend has no test suite and CI runs no test or lint job** — GitLab CI has a single
  `deploy` stage, one job per branch, each running a script in `bash/`. Verification is manual,
  and on the current dev machine a full typecheck/build does not even run (see `repos/web.md`
  § Traps).
- Detail per side: [`repos/api.md`](../repos/api.md) and [`repos/web.md`](../repos/web.md); each
  repo's own `.ai/core/architecture.md` holds its module-by-module breakdown.
