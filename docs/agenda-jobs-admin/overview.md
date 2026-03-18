# Agenda Jobs Admin Page — Overview

## System Context

Rocket.Chat is a large-scale, self-hosted communication platform built on Node.js + MongoDB. Its backend relies on **scheduled background jobs** managed through a standalone **reimplementation** of the Agenda scheduling pattern at `packages/agenda/`. This is NOT a fork of the upstream `agenda/agenda` npm package — it is a purpose-built scheduler sharing the same API surface, tightly integrated with Rocket.Chat's MongoDB connection via `MongoInternals`.

> **Historical note**: Before the `@rocket.chat/agenda` package, Rocket.Chat used SyncedCron. The logger name `Logger('SyncedCron')` still appears in `apps/meteor/server/startup/cron.ts:11` — a direct artifact of this migration.

## Current Job Inventory

| Job Name | Schedule | Type | Purpose | Source File |
|---|---|---|---|---|
| `Generate and save statistics` | `12 <hour> * * *` | `single` | Workspace analytics + license enforcement via `AirGappedRestriction.computeRestriction` | `server/cron/usageReport.ts` |
| `VideoConferences` | `0 */3 * * *` | `single` | Conference cleanup / housekeeping | `server/cron/videoConferences.ts` |
| `Cleanup OEmbed cache` | `24 2 * * *` | `single` | Link preview cache eviction | `server/cron/oembed.ts` |
| `UserDataDownload` | `*/<freq> * * * *` | `single` | GDPR data export processing | `server/cron/userDataDownloads.ts` |
| `NPS` | `21 15 * * *` | `single` | Net Promoter Score survey triggers | `server/cron/nps.ts` |
| `temporaryUploadCleanup` | `31 * * * *` | `single` | Temp file garbage collection | `server/cron/temporaryUploadsCleanup.ts` |
| `calendar-reminders` | Timestamp-driven | `once` | Calendar event push notifications | `server/services/calendar/service.ts` |
| `calendar-status-scheduler` | Timestamp-driven | `once` | Calendar status updates | `server/services/calendar/service.ts:218` |

The Calendar service dynamically schedules jobs via `addAtTimestamp()`. These are `type: 'once'` jobs — each creates a separate document. Active deployments with heavy Calendar usage accumulate **thousands of one-off documents** in `rocketchat_cron`.

## How Agenda Is Used Internally

Rocket.Chat wraps Agenda in the singleton class `AgendaCronJobs` (`packages/cron/src/index.ts`). The exported instance `cronJobs` provides:

```
cronJobs.start(mongo.db)              → initializes Agenda (defaultConcurrency: 1, processEvery: '1 minute')
cronJobs.add(name, cron, fn)          → define + schedule a repeating job (→ type: 'single')
cronJobs.addAtTimestamp(name, date, fn) → schedule a one-off job (→ type: 'once')
cronJobs.remove(name)                 → cancel a job
cronJobs.has(name)                    → check if a job is in the reservedJobs queue
```

**Initialization order**: Jobs registered before `start()` is called are queued in `reservedJobs[]` (private in-memory array). They are flushed into Agenda when `start()` completes. The `has()` method checks this queue, NOT the MongoDB collection.

Every job handler is wrapped in `runCronJobFunctionAndPersistResult()` (`packages/cron/src/index.ts:9–40`), which writes execution start/finish records to the `cron_history` collection (`CronHistoryRaw` model). Agenda itself persists scheduling state to `rocketchat_cron`.

The 1-minute `processEvery` interval means all scheduling has ≤ 1 minute of latency from `nextRunAt` to actual execution start.

## Current Limitations

Despite execution logging infrastructure (`cron_history`), **there is no admin-facing interface to observe or control these jobs**:

1. **Zero observability**: Admins must connect to MongoDB directly to see job status. No UI shows which jobs exist, their state, or failure counts.

2. **Silent failures**: When `Generate and save statistics` fails, `failReason` and `failCount` are written to `rocketchat_cron`. A `Logger('Cron')` error event fires — but production monitoring rarely captures per-job granularity.

3. **No manual trigger**: If `VideoConferences` fails at 09:00, the next run is 12:00. No admin recovery path exists.

4. **Invisible stuck jobs**: Jobs that crash mid-execution retain `lockedAt` until `lockLifetime` (10 min) expires. During disrupted deploys, zombie-locked jobs are invisible.

5. **Orphaned `cron_history`**: The collection logs every execution but has no consumer in the admin UI, no TTL index — it grows unboundedly.

6. **`reservedJobs` queue invisible**: Jobs queued before scheduler start are stored in-memory only — there is no persistence or visibility into this pre-flight state.

## Why This Matters

In Rocket.Chat Enterprise and Cloud deployments, the job scheduler underpins:

- **Licensing**: `Generate and save statistics` → `sendUsageReport` → `AirGappedRestriction.computeRestriction`
- **GDPR compliance**: `UserDataDownload` processes pending data export requests
- **User trust**: `calendar-reminders` misfire = silent appointment notification failure
- **Platform stability**: `temporaryUploadCleanup` prevents disk exhaustion from orphaned uploads

Failures in any of these are production incidents. Without observability, support teams rely on end-user reports — the slowest possible signal.

## Scope of This Project

This GSoC project delivers a first-class admin page at `/admin/cron-jobs` that:

- **Lists** all jobs from `rocketchat_cron` with live status (scheduled, running, failed, disabled, stuck) — default view groups by job name via aggregation
- **Shows execution history** from `cron_history` per job in a `ContextualbarDialog` drawer
- **Enables admin actions**: trigger now, disable, enable, unlock stuck jobs — with confirmation modals and toast feedback
- **Exposes 7 REST endpoints** under `API.v1` following existing route conventions
- **Registers types** in `packages/rest-typings/src/v1/cron-jobs/` for end-to-end type safety with `useEndpoint()`
- **Uses AJV validators** (not Zod) matching the existing `rest-typings` validation framework
- **Adds dual permissions**: `view-cron-jobs` (read) + `manage-cron-jobs` (write)
- **Extends `AgendaCronJobs`** with 3 public methods (`hasDefinition`, `triggerNow`, `getRegisteredJobNames`) — preserving private `scheduler` encapsulation
- **Adds TTL index** to `cron_history` for automatic 30-day cleanup
- **Adds supplementary indexes** to `rocketchat_cron` for admin query patterns (sparse failCount index, name+nextRunAt compound, disabled+name compound)
- **Handles dynamic job names** (e.g., `calendar-status-scheduler`) via regex-based name filtering with ReDoS prevention
- **Provides i18n keys** in `packages/i18n/src/locales/en.i18n.json` with automatic fallback for other locales
- **UI follows Rocket.Chat patterns**: `Page`/`PageHeader`/`PageContent` from `@rocket.chat/ui-client`, `ContextualbarDialog` for detail drawer, `GenericTable` for list, `Pagination` for paging
