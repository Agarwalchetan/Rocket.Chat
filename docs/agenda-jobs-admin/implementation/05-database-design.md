# 05 — Database Design

## Collections Overview

| Collection | Owner | Read By | Write By |
|---|---|---|---|
| `rocketchat_cron` | `@rocket.chat/agenda` (Agenda) | `AgendaJobsRaw` (new) | Agenda only |
| `cron_history` | `CronHistoryRaw` | `CronHistory.findPaginated()` (new usage) | `runCronJobFunctionAndPersistResult()` |

---

## `rocketchat_cron` Schema

`_id` is `ObjectId` — Agenda creates documents with native BSON ObjectId, not string.

```ts
interface IAgendaJob {
  _id: ObjectId;             // Mongo-native BSON ObjectId
  name: string;              // Human-readable job name
  type: 'once' | 'single' | 'normal';
                             // 'single': one doc per job (created by scheduler.every())
                             // 'normal': one doc per occurrence
                             // 'once': one-time (created by scheduler.schedule()/now())
  repeatInterval?: string | number;  // cron expression OR ms interval
  repeatTimezone?: string | null;    // IANA timezone string
  repeatAt?: string;                 // human-interval string
  nextRunAt?: Date | null;   // next scheduled execution
  lastRunAt?: Date;          // when last execution started
  lastFinishedAt?: Date;     // when last execution completed
  lockedAt?: Date | null;    // null = available; Date = locked by a worker
  disabled?: boolean;        // true = skipped by scheduler
  priority?: number;         // 0 = default; higher = runs first
  failReason?: string;       // message from last failure
  failCount?: number;        // cumulative failure count
  failedAt?: Date;           // timestamp of last failure
  lastModifiedBy?: string;   // Agenda instance identifier
  data?: Record<string, any>; // arbitrary job payload
}
```

**Example document** (real `NPS` job):
```json
{
  "_id": ObjectId("65a1b2c3d4e5f6a7b8c9d0e1"),
  "name": "NPS",
  "type": "single",
  "repeatInterval": "21 15 * * *",
  "nextRunAt": ISODate("2026-03-19T15:21:00.000Z"),
  "lastRunAt": ISODate("2026-03-18T15:21:00.000Z"),
  "lastFinishedAt": ISODate("2026-03-18T15:21:03.412Z"),
  "lockedAt": null,
  "disabled": false,
  "priority": 0,
  "failCount": 0,
  "lastModifiedBy": null
}
```

---

## `cron_history` Schema

```ts
interface ICronHistoryItem {
  _id: string;         // Random.id() — 17-char alphanumeric
  name: string;        // job name
  intendedAt: Date;    // when the wrapper created this record
  startedAt: Date;     // actual execution start
  finishedAt?: Date;   // set on completion
  result?: any;        // return value of the job function
  error?: any;         // error.stack string on failure
}
```

---

## Existing Indexes (Agenda-Managed)

`rocketchat_cron`:
```
{ name: 1, nextRunAt: 1, priority: -1, lockedAt: 1, disabled: 1 }
Named: 'findAndLockNextJobIndex'
```

`cron_history`:
```
{ intendedAt: 1, name: 1 } — unique
```

---

## New Indexes (Added by This Project)

### `rocketchat_cron` — via `AgendaJobsRaw.modelIndexes()`

```ts
[
  // Admin list: sort by name ASC, next run DESC
  { key: { name: 1, nextRunAt: -1 } },

  // Failed job filter: sparse (only indexes docs with failCount)
  { key: { failCount: 1, failedAt: -1 }, sparse: true },

  // Disabled job filter
  { key: { disabled: 1, name: 1 } },
]
```

These are supplementary to Agenda's `findAndLockNextJobIndex`. The `sparse: true` on the failCount index avoids indexing the majority of documents where `failCount` is 0 or undefined.

### `cron_history` — TTL index via `CronHistoryRaw.modelIndexes()`

```ts
{ key: { intendedAt: 1 }, expireAfterSeconds: 60 * 60 * 24 * 30 }
```

Does NOT conflict with the existing `{ intendedAt: 1, name: 1 }` compound unique index — MongoDB handles multiple indexes on the same field independently. The TTL background task runs once per minute.

---

## Aggregation Pipeline: Grouped View

The default admin list shows one row per distinct job name:

```ts
[
  {
    $group: {
      _id: '$name',
      count: { $sum: 1 },
      nextRunAt: { $min: '$nextRunAt' },
      lastRunAt: { $max: '$lastRunAt' },
      lastFinishedAt: { $max: '$lastFinishedAt' },
      totalFailCount: { $sum: { $ifNull: ['$failCount', 0] } },
      isDisabled: { $max: { $cond: [{ $eq: ['$disabled', true] }, 1, 0] } },
      hasLocked: {
        $max: {
          $cond: [{ $ne: ['$lockedAt', null] }, 1, 0],
        },
      },
    },
  },
  { $sort: { _id: 1 } },
]
```

**Output shape** (`JobSummary`):
- `_id`: job name (string, the `$group` key)
- `count`: number of documents with this name
- `nextRunAt`, `lastRunAt`, `lastFinishedAt`: date extremes
- `totalFailCount`: sum of all fail counts
- `isDisabled`, `hasLocked`: 0 or 1 (boolean-like)

For 6 core Rocket.Chat jobs: 6 rows. For deployments with thousands of calendar jobs: still only 7–8 rows (calendar-reminders + calendar-status count as 1–2 groups).

---

## Status Derivation Logic

Status is **computed**, not stored in MongoDB:

```ts
function deriveJobStatus(job: Partial<IAgendaJob>, lockLifetimeMs = 600_000): AgendaJobStatus {
  if (job.disabled) return 'disabled';
  if (job.lockedAt) {
    const lockedMs = Date.now() - new Date(job.lockedAt).getTime();
    return lockedMs > lockLifetimeMs ? 'stuck' : 'running';
  }
  if ((job.failCount ?? 0) > 0) return 'failed';
  return 'scheduled';
}
```

This is a pure function used in both the backend (for `buildMongoFilter()`) and the frontend (`CronJobStatusBadge.tsx`).

---

## Index Utilization Matrix

| Query Pattern | Index Hit | Used By |
|---|---|---|
| List all jobs sorted by name | `{ name: 1, nextRunAt: -1 }` | `GET /cron-jobs` |
| Filter: failed jobs | `{ failCount: 1, failedAt: -1 }` (sparse) | `status=failed` filter |
| Filter: disabled jobs | `{ disabled: 1, name: 1 }` | `status=disabled` filter |
| Find one job by name prefix | `{ name: 1, nextRunAt: -1 }` (prefix) | `findOneByName` |
| History by job name | `{ intendedAt: 1, name: 1 }` (existing) | `GET /cron-jobs/:id/history` |
| TTL cleanup | `{ intendedAt: 1 }` (new TTL) | MongoDB background task |
| Scheduler locking | `findAndLockNextJobIndex` (existing) | Agenda internal — NOT admin |
