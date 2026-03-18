# 01 — System Analysis

## Agenda Package Location and Initialization

**Package**: `packages/agenda/` — Rocket.Chat's standalone **reimplementation** of the Agenda scheduling pattern (NOT a fork of `agenda/agenda` on npm — no git fork relationship exists).

**Entry point**: `packages/agenda/src/index.ts` exports `{ Agenda, Job }`.

**Wrapper singleton**: `packages/cron/src/index.ts`

```ts
// packages/cron/src/index.ts (line 56–200)
export class AgendaCronJobs {
  private scheduler: Agenda | undefined;

  public async start(mongo: Db): Promise<void> {
    this.scheduler = new Agenda({
      mongo,
      db: { collection: 'rocketchat_cron' },
      defaultConcurrency: 1,
      processEvery: '1 minute',
    });
    // ... event listeners
    await this.scheduler.start();
    // flush reservedJobs
  }

  public async add(name, schedule, callback) { /* ... */ }
  public async addAtTimestamp(name, when, callback) { /* ... */ }
  public async remove(name) { /* ... */ }
  public async has(jobName) { /* ... */ }
}

export const cronJobs = new AgendaCronJobs();
```

**Startup entry**: `apps/meteor/server/cron/start.ts`

```ts
import { cronJobs } from '@rocket.chat/cron';
import { MongoInternals } from 'meteor/mongo';

export const startCron = async () => {
  return cronJobs.start(
    (MongoInternals.defaultRemoteCollectionDriver().mongo as any).client.db()
  );
};
```

Called by `startCronJobs()` in `apps/meteor/server/startup/cron.ts`, which is called during server startup.

---

## Current Job Definitions (Complete Inventory)

| Job Name | Schedule | File | Notes |
|---|---|---|---|
| `Generate and save statistics` | `12 <currentHour> * * *` | `server/cron/usageReport.ts` | Dynamic at startup; feeds license enforcement |
| `VideoConferences` | `0 */3 * * *` | `server/cron/videoConferences.ts` | Runs every 3 hours |
| `Cleanup OEmbed cache` | `24 2 * * *` | `server/cron/oembed.ts` | Daily at 02:24 |
| `UserDataDownload` | `*/<freq> * * * *` | `server/cron/userDataDownloads.ts` | Freq = setting `UserData_ProcessingFrequency` |
| `NPS` | `21 15 * * *` | `server/cron/nps.ts` | Daily at 15:21 |
| `temporaryUploadCleanup` | `31 * * * *` | `server/cron/temporaryUploadsCleanup.ts` | Hourly at :31 |
| `calendar-reminders` | Timestamp | `server/services/calendar/service.ts` | One per-user event (dynamic) |
| `calendar-status-scheduler` | Timestamp | `server/services/calendar/service.ts:218` | Dynamic per calendar entry (not `calendar-status-<id>` — verified via grep: `const schedulerJobId = 'calendar-status-scheduler'`) |

Pattern for `UserDataDownload`: re-scheduled dynamically via `settings.watchMultiple()`, calling `cronJobs.remove()` then `cronJobs.add()` when the processing frequency setting changes.

---

## MongoDB Collections

### `rocketchat_cron`

Created and managed by Agenda. Contains one document per registered job (for `type: 'single'`) or one document per scheduled occurrence (for `type: 'normal'`).

**Document structure** (derived from `IJob` in `packages/agenda/src/definition/IJob.ts`):

```ts
interface IAgendaJob {
  _id: ObjectId;            // Mongo-generated
  name: string;             // job name string, e.g. 'NPS'
  type: 'once' | 'single' | 'normal';
  repeatInterval?: string | number; // cron string or ms number
  repeatTimezone?: string | null;
  repeatAt?: string;        // human-interval string for repeat-at-time jobs
  nextRunAt?: Date | null;  // computed by Job.computeNextRunAt()
  lastRunAt?: Date;
  lastFinishedAt?: Date;
  lockedAt?: Date | null;   // null = unlocked, Date = locked by a worker
  disabled?: boolean;
  priority?: number;        // default 0
  failReason?: string;      // last failure message
  failCount?: number;       // cumulative failure count
  lastModifiedBy?: string;  // name of the Agenda instance
  data?: Record<string, any>;
}
```

**Existing index** (created by Agenda at `dbInit()`):
```
{ name: 1, nextRunAt: 1, priority: -1, lockedAt: 1, disabled: 1 }
Named: 'findAndLockNextJobIndex'
```

### `cron_history`

Created and managed by `CronHistoryRaw` (`packages/models/src/models/CronHistoryModel.ts`). Written to by `runCronJobFunctionAndPersistResult()` in `packages/cron/src/index.ts`.

**Document structure** (from `ICronHistoryItem` in `packages/core-typings/src/ICronHistoryItem.ts`):

```ts
interface ICronHistoryItem {
  _id: string;        // Random.id()
  name: string;       // job name
  intendedAt: Date;   // when the record was created (approximately when the job was triggered)
  startedAt: Date;    // actual execution start
  finishedAt?: Date;  // set on completion
  result?: any;       // return value of the job function
  error?: any;        // error.stack string on failure
}
```

**Existing index**: `{ intendedAt: 1, name: 1 }` (unique) — intended to deduplicate concurrent runs.

---

## Internal Service Usage

All job handlers are called inside `AgendaCronJobs.define()`:

```ts
private async define(jobName: string, callback): Promise<void> {
  this.scheduler.define(jobName, async () => {
    await runCronJobFunctionAndPersistResult(async () => callback(), jobName);
  });
}
```

Event listeners in `AgendaCronJobs.start()`:
- `start`: debug log with `jobId`, `jobName`, `nextRunAt`
- `complete`: info log with duration calc
- `success`: debug log
- `fail`: **error log** with `failCount` and `failReason`
- `error:database`: error log
- `error`: error log

These logs go to `Logger('Cron')` — only visible in server logs, not in the admin UI.

---

## What Is Missing

1. No `listJobs()` or `findPaginated()` surface on `AgendaJobsRaw` (model doesn't exist yet)
2. No `CronJobsAdminService` to orchestrate read/write actions
3. No REST endpoints under `/api/v1/cron-jobs/*`
4. No admin route or UI component
5. No `view-cron-jobs` / `manage-cron-jobs` permissions (need split read/write)
6. `cron_history` has no TTL index — collection grows indefinitely
7. No endpoint type declarations in `packages/rest-typings/` — `useEndpoint()` would have no type resolution
8. `AgendaCronJobs` has no public methods to check definitions or trigger jobs — `scheduler` is `private` (line 59)
9. No i18n keys for the admin page UI labels
10. `Page`, `PageHeader`, `PageContent` must be imported from `@rocket.chat/ui-client` (not `@rocket.chat/fuselage`)
