---
title: >-
  ADR 0005 — Transcoder callbacks are deduped by a Redis claim; CAS stays the
  correctness boundary
summary: >-
  API-SCOPED. The transcoder→CMS leg is Redis Pub/Sub, which broadcasts to every
  API instance. A Redis SET NX claim at each listener's entry point makes one
  instance handle each event; compare-and-set on the row remains the only thing
  guaranteeing correctness. Rules every new dubbing listener must follow.
tags:
  - architecture
  - api
  - ai-dubbing
  - queues
  - redis
  - agent-rule
status: ready
links:
  - contracts/ai-dubbing.md
  - repos/api.md
updated: '2026-09-30'
---
# ADR 0005 — Transcoder callbacks are deduped by a Redis claim; CAS stays the correctness boundary

**Date:** 2026-09-30
**Scope:** API repo, `modules/content/ai_dubling`
**Status:** Accepted

## Context

The CMS↔transcoder pipeline uses two mechanisms over the same Redis:

- **CMS → transcoder:** BullMQ queues. CMS is producer only.
- **transcoder → CMS:** **Redis Pub/Sub**, channel name identical to the queue name
  (`ChannelResolverService` falls through to `default: return queue` for the five dubbing v2 queues).

Redis Pub/Sub is a broadcast with no ack and no redelivery. Production runs **6 API instances**
against **1 transcoder**, so one `PUBLISH` was handled 6 times. BullMQ's `@OnQueueEvent` — kept as a
fallback path in the same listeners — also delivers to all 6.

The design assumed compare-and-set (`UPDATE ... WHERE id=? AND status IN (...)`, `affected > 0`)
would absorb the fan-out. It does, but only for work placed **after** a CAS whose result is read.
Everything before it multiplied by 6: file logs, `job_step_log` rows, `cleanup-storage` (which
changes no status and so has no CAS at all), and — because `dispatchTranslate` discarded the CAS
boolean — 6 HTTP requests per Language Task to the AI partner.

## Decision

1. **One claim per event, at the listener's entry point.** `EventDedupService.runOnce(channel,
   jobId, event, handler)` does `SET dubbing:evt:{channel}:{jobId}:{event} <instance> PX 60000 NX`.
   The winner runs the handler; the others return immediately.
2. **Both delivery paths take the same claim**, with `event` lowercased in the key, because Pub/Sub
   sends the transcoder's enum spelling (`DOWNLOADING`) while `@OnQueueEvent` calls in with
   `downloading`. They must collapse onto one key to cancel each other out.
3. **The claim is not a correctness boundary.** It is released (`DEL`) when the handler throws, and
   a Redis failure runs the handler anyway rather than dropping the event. Turning at-least-once
   into at-most-once is the worse failure here — Pub/Sub never redelivers.
4. **CAS remains the guarantee, and its result must always be read.** Any side effect that must not
   happen twice goes behind `if (won)`.
5. **Fan-out to children is never gated on the parent's CAS.** It re-reads the parent's state and
   lets each child claim itself, so an instance dying mid-loop does not strand the remaining
   children — the next delivery re-runs the loop harmlessly.
6. **`@Cron` also runs on all instances** and takes the same claim (channel `dubbing-cron`), with a
   claim TTL shorter than the cron period.

## Rules for new listeners in this module

- Wrap `on('message')` **and** every `@OnQueueEvent` handler in `runOnce`, using the queue name as
  channel and the transcoder's event name (lowercased by the service).
- Read every `updateStatusWithCAS` result. A discarded boolean is the defect this ADR exists for.
- Keep the `expected` sets a one-way ladder (each transition accepts only states before it). Pub/Sub
  handlers are not awaited by ioredis, so two events of the same job run concurrently in one
  instance and can finish out of order; the ladder is what stops a stale event regressing a status.
- CAS cannot tell you whether anyone *finished* the work. Anything that can strand an entity needs a
  reconciliation sweep, not just a CAS.

## Consequences

- A stuck-task sweep in `ReconciliationCron` re-activates tasks left `CREATED` after their job
  extracted audio, and fails tasks sitting in `AI_TRANSLATING` past a wide threshold. It does not
  re-dispatch: a slow AI server is indistinguishable from a dead one.
- Because that sweep can mark a live task `FAILED`, `handleCallback` accepts `FAILED` in the CAS set
  of its success branch, so a late success still rescues the task. `REJECTED` and `DELETED_BY_USER`
  stay out of that set — a late callback must not override a human decision.
- The rule for activating a `CREATED` task (the softsub/hardsub split) lives in exactly one place,
  `LanguageTaskService.activateCreatedTask`; it had drifted into three copies.
- The claim is best-effort, so duplicate audit rows remain possible after a Redis restart. Accepted.
