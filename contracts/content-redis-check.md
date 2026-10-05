---
title: Content Redis check — API contract
summary: >-
  GET /common/check_content_redis compares a content's DB record with its value
  on every Redis server (REDIS_TYPE = 3). Reworked 2026-10-05 — breaking
  response change, FE and API must ship together.
tags:
  - contract
  - api
  - frontend
  - redis
status: ready
links:
  - repos/api.md
  - repos/web.md
updated: '2026-10-05'
---
# Content Redis check — `GET /common/check_content_redis`

Used by the CMS "Kiểm tra thông tin redis" (check Redis data) dialog on the VOD, movie and Music
Clip edit forms. FE: `modules/content/ContentRedisCheck` (one dialog for all three kinds).
API: `modules/common` — `check-content-redis.service.ts` (no interface files, concrete DI).

## Request (query, validated)

| Param | Rule | Error (400, `message` is an **array** of strings) |
|---|---|---|
| `CONTENT_ID` | integer ≥ 1 | `CONTENT_ID phải là số nguyên`, `CONTENT_ID phải lớn hơn 0` |
| `CONTENT_KEY` | `CONTENT` \| `MOVIE` \| `MUSIC_VIDEO` | `CONTENT_KEY phải là một trong: …` |

Content not found → 404 `Nội dung không tồn tại`. No Redis server configured → 404.

## Per kind

| `CONTENT_KEY` | Redis key | Info field in the Redis value | DB tables |
|---|---|---|---|
| `CONTENT` (VOD) | `content_detail_{id}` | `info_content` | CONTENT, CONTENT_TRAILER.CONTENT_ID, CONTENT_SERIES.CONTENT_ID |
| `MOVIE` | `movie_detail_{id}` | `info_movie` | MOVIE, MOVIE_TRAILER.MOVIE_ID, MOVIE_SERIES.MOVIE_ID |
| `MUSIC_VIDEO` | `music_video_detail_{VIDEO_ID}` | `info_music` | MUSIC_VIDEO, MUSIC_VIDEO_TRAILER.CONTENT_ID, MUSIC_VIDEO_SERIES.VIDEO_ID |

All three Redis values share the shape `{ <info field>, partition_list, is_favorite, redis_updated_at }`;
they are written by the external redis-api (`/v8/apiredis/*-detail-write`), not by the CMS.
`MUSIC_VIDEO.PROVIDER_ID` does not exist in the DB — redis-api fills it — so it is skipped when
comparing (`derivedFields`). Movie values carry no `redis_updated_at`.

## Response `data`

```ts
{
  redis_key: string;
  redis_results: { redis_id: number; redis_name: string; has_key: boolean; redis_updated_at: string | null }[];
  // only servers that differ; a server in redis_results but absent here matches the DB
  compare_results: {
    redis_id: number; redis_name: string;
    // string = key missing / server unreachable; array items: field diff, or a string when the info field is missing
    info_differences?: string | ({ field: string; redis_value: unknown; db_value: unknown } | string)[];
    // a partition may carry only `error` (in Redis, not in DB) — no `differences`
    partition_differences?: { partition: unknown; differences?: { field; redis_value; db_value }[]; error?: string }[];
  }[];
}
```

**Breaking change (2026-10-05):** `content` (full DB row), `redis_results[].redis_value` (full
Redis value) and `redis_host` / `redis_port` were removed (payload went from 6–20 KB to <1 KB).

## Behaviour guarantees

- `CONTENT_ID` reaches SQL only as a bound parameter (it used to be string-interpolated).
- Each Redis connection is closed in `finally`; 5 s connect timeout, no retry. One unreachable
  server yields `info_differences: "Không kết nối được tới Redis server này"` for that server only.
