---
title: AI Dubbing — API contract
tags:
  - contract
  - api
  - ai-dubbing
summary: >-
  Endpoint contract for the AI Dubbing module (video-jobs, language-tasks,
  final-outputs, job-step-logs): routes, payloads, status machines, and the
  traps the frontend must code around.
status: ready
links:
  - conventions.md
  - repos/api.md
  - architecture/decisions/0001-new-feature-module-conventions.md
updated: '2026-09-04'
---
# AI Dubbing — API contract

**Backend source**: `src/modules/content/ai_dubling/` in `api-smart-cms` (module class
`AiDublingModule`, registered in `src/app.module.ts`). Note the folder is spelled `ai_dubling`
(missing `b`) — the domain word is **dubbing**.

Read [`conventions.md`](../conventions.md) Part 1 first — the response envelope and route prefix
below are the shared ones, not module-specific.

## What the feature does

An operator points the CMS at a **source video file on FTP A** and asks for one AI-dubbed
(voice-over) language track. The backend orchestrates a pipeline across three executors —
`cms` (this API), `transcoder` (BullMQ workers, a separate service), `ai_server` (VNPT BigData) —
and the operator reviews a **preview**, approves or rejects it, then combines approved tracks into
a **multi-audio final output** file.

Three aggregates, in a strict parent→child chain:

```
VideoJob (1 source video)
   └─ LanguageTask (1 per target language; audio track + preview + human decision)
   └─ FinalOutput  (multi-audio muxed result; max 2 alive per VideoJob)
JobStepLog — flat audit trail over all three (entityType + entityId)
```

## Route prefix — two `api/v1` segments, not one

The global prefix is `BASE_URL` (deployed `/api/v1/cms-api`) **and** each controller declares its
own `api/v1/...` path. The real URLs are therefore doubled:

```
POST https://<host>/api/v1/cms-api/api/v1/video-jobs
```

This is not a typo: the backend builds its own AI-callback URL the same way
(`ai-integration.service.ts` → `${domain}/api/v1/cms-api/api/v1/language-tasks/callback`). The
frontend's `BaseService` baseURL already contains the first `api/v1/cms-api`, so service paths
must start `api/v1/video-jobs`, **not** `video-jobs`.

## Auth & RBAC

- Global `AuthGuard` (`APP_GUARD`), **JWT Bearer required on every route in this module** — the
  AI-server callback route included (see Caveats).
- RBAC route names via `@RouteInfo`. Menu keys / permission names the frontend menu must map:

| menu_key | route names |
|---|---|
| `ai-dubbing-video-jobs` | `.list` `.create` `.detail` `.retry` `.language-tasks` `.final-outputs` `.create-final-output` |
| `ai-dubbing-language-tasks` | `.list` `.detail` `.delete` `.approve` `.reject` `.retry` `.callback` |
| `ai-dubbing-final-outputs` | `.list` `.detail` `.delete` |
| `ai-dubbing-job-step-logs` | `.list` |

## Endpoints

All paths below are relative to the doubled prefix above. All wrap the standard envelope
(`{status, result, message, data, error}`).

### Video Jobs — `api/v1/video-jobs`

| Method | Path | Body / Query | `data` | HTTP |
|---|---|---|---|---|
| GET | `/` | `VideoJobFilterDto` | `{items, total, page, limit}` | 200 |
| POST | `/` | `CreateVideoJobDto` | `VideoJob` | **201** |
| GET | `/:id` | — | `VideoJob` \| `null` | 200 |
| POST | `/:id/retry` | — | `null` | **202** |
| GET | `/:id/language-tasks` | `status` only | **plain array** of `LanguageTask` | 200 |
| GET | `/:id/final-outputs` | (ignored) | **plain array** of `FinalOutput` | 200 |
| POST | `/:id/final-outputs` | `CreateFinalOutputDto` | `FinalOutput` | **201** |

`CreateVideoJobDto` — one request creates **one** VideoJob **and exactly one** LanguageTask:

```jsonc
{
  "sourceLang": "vi",            // required, ≤16
  "targetLang": "en",            // required, ≤16 — the single language task created
  "subType": 1,                  // optional int. 1 = softsub/voiceover, 2 = hardsub. Persisted default 0
  "speaker": "en",               // optional ≤20; defaults to targetLang
  "delay": 0,                    // optional number (seconds)
  "shortedSub": 0,               // optional int, 0 off / 1 on
  "metaData": {                  // required
    "content_id": 1024,          // required (number|string)
    "series_id": 2048,           // required (number|string)
    "type_id": 1,                // required (number|string)
    "path": "/Wistoria/xxx.mkv", // required — source video on FTP A, ≤1024
    "subtitlePath": "/subs/x.srt"// optional, ≤1024
  }
}
```

There is **no multi-language create endpoint.** `CreateLanguageTasksDto` (`targetLangs: string[]`)
exists in `dto/` but is referenced nowhere in the repo — dead code, do not build a UI against it.
To dub N languages, POST N video jobs today.

`createdBy` is taken server-side from `req.user.email ?? req.user.username ?? 'admin'` — never send
it.

`CreateFinalOutputDto`:

```jsonc
{ "tracks": [ { "taskId": 5001, "order": 1 }, { "taskId": 5002, "order": 2 } ] }
```

Every `taskId` must belong to that VideoJob **and** be `APPROVED`, else 400. Max **2 alive**
FinalOutputs per VideoJob (alive = status not in `DELETED`/`FAILED`), enforced under a pessimistic
row lock → 409 `"Đã đủ 2 bản Final Output đang hoạt động..."`. The UI must offer "delete one to
free a slot" on that error.

### Language Tasks — `api/v1/language-tasks`

| Method | Path | Body | `data` | HTTP |
|---|---|---|---|---|
| GET | `/` | `LanguageTaskFilterDto` | `{items, total, page, limit}` | 200 |
| GET | `/:id` | — | `LanguageTask` \| `null` | 200 |
| DELETE | `/:id` | — | `null` | **204** — only from `APPROVED` |
| POST | `/:id/approve` | — | updated `LanguageTask` | 200 — only from `PREVIEW_READY` |
| POST | `/:id/reject` | `{reason?}` ≤500 | updated `LanguageTask` | 200 — only from `PREVIEW_READY` |
| POST | `/:id/retry` | — | `null` | **202** — only from `FAILED` |
| POST | `/callback` | `UpdateAudioAIDto` | `null` | 200 — **AI server → CMS, not for the UI** |

`reject`'s `reason` is stored in the task's `errorMessage` column and also written to a
`reject-preview` job step log.

### Final Outputs — `api/v1/final-outputs`

| Method | Path | `data` | HTTP |
|---|---|---|---|
| GET | `/` | `{items, total, page, limit}` | 200 |
| GET | `/:id` | `FinalOutput` \| `null` | 200 |
| DELETE | `/:id` | `null` | **204** — soft delete → `DELETED`, frees a slot; idempotent |

### Job Step Logs — `api/v1/job-step-logs`

`GET /` with `entityType` (`video_job` \| `language_task` \| `final_output`), `entityId`, plus
pagination → `{items, total, page, limit}`, newest first. This is the "why did it fail / what
happened" drawer for a detail screen.

### Pagination query (all list endpoints)

`page` (≥1, default 1), `limit` (1..100, default 20), and an undocumented `pageSize` that
**overrides `limit`** when present. `ValidationPipe({whitelist: true})` strips unknown params
silently — a typo'd filter is not an error, it just does nothing.

## Status machines (drive the UI)

`VideoJobStatus` — `CREATED → DOWNLOADING → EXTRACTING_AUDIO → AUDIO_EXTRACTED`, or `FAILED` from
any of the first three. Only `FAILED` allows `POST /:id/retry`.

`LanguageTaskStatus`:

```
CREATED → AI_TRANSLATING → AI_COMPLETED → PREVIEW_GENERATING → PREVIEW_READY
                                                                    ├→ APPROVED → DELETED_BY_USER
                                                                    ├→ REJECTED
                                                                    └→ EXPIRED  (cron, 15 days)
FAILED  ← from any pipeline stage; retryable
```

- **Approve/Reject are enabled only in `PREVIEW_READY`.** Any other status → 409 with the current
  status in the message.
- **Delete is enabled only in `APPROVED`** (→ `DELETED_BY_USER`, frees NFS space).
- **Retry is enabled only in `FAILED`.** Server-side, retry only actually re-enqueues when
  `translatedAudioPath` is set (preview-stage failure). A translate-stage failure hits an empty
  `if` branch — the call returns 202 but **nothing is re-dispatched**. Do not promise the operator
  it worked; surface the task status instead.
- `PREVIEW_READY` starts a **15-day** clock (`previewReadyAt`); an hourly cron flips untouched
  tasks to `EXPIRED` with `decision = expired`. Show the deadline/countdown on the review screen.

`decision` (`approved` \| `rejected` \| `expired`, lowercase — unlike every other enum here, which
is UPPERCASE) plus `decisionAt` record the human verdict.

`FinalOutputStatus` — created directly as `QUEUED → MERGING → COMPLETED`, or `FAILED`; `DELETED` on
soft delete. `SELECTING` (the column default) and `BLOCKED` are declared in the enum but nothing
ever sets them — treat as legacy/unreachable, still handle defensively in a status map.

## Response shapes

The endpoints return **raw TypeORM entities** — no response DTO, no field filtering. Fields as
persisted:

- `VideoJob`: `id, sourceFilePath, sourceLang, status, extractedAudioPath, checksumMd5,
  errorMessage, createdBy, subType, metaData, createdAt, updatedAt`.
  `sourceFilePath` is **null at creation** — the transcoder fills it in on `AUDIO_EXTRACTED`; the
  submitted FTP path lives in `metaData.path` until then. A list column should read
  `sourceFilePath ?? metaData?.path`.
- `LanguageTask`: `id, videoJobId, targetLang, status, uploadedSubtitlePath, translatedAudioPath,
  shortedSubtitlePath, previewUrl, previewReadyAt, decision, decisionAt, speaker, delay,
  errorMessage, createdAt, updatedAt, shortedSub`.
  `previewUrl` is the player source and is only populated at `PREVIEW_READY`.
- `FinalOutput`: `id, videoJobId, tracks ([{taskId, order}]), status, outputPath, errorMessage,
  createdBy, createdAt, updatedAt`. `outputPath` = the muxed file on FTP A, set at `COMPLETED`.
- `JobStepLog`: `id, entityType, entityId, stepName, executor, status, requestPayload,
  responsePayload, errorMessage, startedAt, finishedAt`. `stepName` values seen:
  `download-video`, `extract-audio`, `translate`, `generate-preview`, `merge-final`,
  `reject-preview`, `preview-expiry-check`. `executor` ∈ `cms | transcoder | ai_server`;
  `status` ∈ `STARTED | SUCCESS | FAILED`.

## Idempotency

`POST /video-jobs`, `POST /video-jobs/:id/final-outputs`, `POST /language-tasks/:id/approve` and
`.../reject` accept an optional **`Idempotency-Key`** header (UUID v4). Behaviour:

- Absent → no protection at all. Send one for every create/decision to survive double-click and
  retry-on-timeout.
- Same key + same body → the **cached response is replayed**, handler not re-run.
- Same key + **different** body → 409 `"Idempotency-Key đã được sử dụng với payload khác..."`.
  Generate a fresh UUID per distinct user action, not per session or per screen.
- The key is scoped by `METHOD + full URL`, so it does not collide across endpoints.

## Errors

Failures come back through the same envelope with `result: -1` and the message in `message`. The
messages are **Vietnamese and user-presentable** — show them rather than inventing your own. Codes
in play: 400 (invalid track / bad callback payload), 401 (missing or bad JWT), 404 (id not found),
409 (wrong status for the action, slot limit reached, idempotency-key reuse, lost CAS race).

409 from a CAS race — `"Trạng thái của Task đã bị thay đổi bởi thao tác khác"` — means another
operator or the pipeline moved the record underneath the user. Refetch the record and re-render
rather than retrying the mutation.

## Caveats — verified in code, plan around them

1. **Declared filters that do nothing.** `videoJobId` on `LanguageTaskFilterDto` /
   `FinalOutputFilterDto` and `sourceFilePath` on `VideoJobFilterDto` are validated and then
   **dropped** — the repositories' `findAndCount` only ever applies `status`. To list a job's
   children, use the nested routes `GET /video-jobs/:id/language-tasks` and `.../final-outputs`.
2. **The nested routes are unpaginated.** They return a bare array in `data` (ascending
   `createdAt` for tasks, descending for outputs), not `{items,total,...}`. `status` filters only
   on the language-tasks route; the final-outputs route ignores its query entirely.
3. **`bigint` ids.** `id`, `videoJobId`, `entityId` are TypeORM `bigint` and `delay` is `decimal`,
   which the MySQL driver hands back as **strings** with no transformer in place. Compare with
   `String(a) === String(b)` or coerce on receipt; do not `===` an id against a number. Worth a
   quick check against a real response before building on it.
4. **204 responses carry a body.** `TransformInterceptor` sets the HTTP status from the envelope,
   so the three delete/`NO_CONTENT` endpoints emit 204 *with* an envelope. Fastify may drop that
   body — treat 204 as success without parsing.
5. **`POST /language-tasks/callback` sits behind the global `AuthGuard`** with no `@Public()`. It
   is the AI server's callback URL, so either the partner sends a JWT or this route 401s in
   practice. Not a frontend concern, but it explains dubbing runs stuck at `AI_TRANSLATING`.
6. **No polling endpoint and no websocket.** Progress arrives via BullMQ/Redis into the DB; the UI
   must **poll** `GET /video-jobs/:id` + the nested task list while any status is non-terminal.
7. **No upload endpoint here.** `metaData.path` / `subtitlePath` are paths on FTP A that must
   already exist; the source-file picker is somebody else's API.
8. **Queue retries are disabled** (`attempts: 1` everywhere, backoff commented out), so a
   transcoder hiccup surfaces as `FAILED` immediately and the operator's manual retry is the only
   recovery path. Make the retry button prominent on failed rows.
9. This module uses `BaseController` and `sendOkResponse`, i.e. it follows the **legacy** pattern,
   not [ADR 0001](../architecture/decisions/0001-new-feature-module-conventions.md). Do not read it
   as an example of current backend conventions — it is only the contract that matters here.

## Screens this contract implies

- **Video Job list** — status filter + pagination; create dialog (the `CreateVideoJobDto` form);
  retry action on `FAILED`.
- **Video Job detail** — job status/progress, its language tasks, its final outputs, and a
  step-log drawer (`job-step-logs?entityType=video_job&entityId=<id>`).
- **Preview review** — player on `previewUrl`, Approve / Reject(+reason) enabled only at
  `PREVIEW_READY`, with the 15-day expiry countdown.
- **Final output builder** — pick approved tasks, order them, POST; show the 2-slot limit and
  offer delete-to-free-a-slot.
