# 03 — Backend Architecture

## Layer Overview

```
HTTP Layer          apps/meteor/app/api/server/v1/cron-jobs.ts
                           ↓
Service Layer       apps/meteor/server/services/cronJobs/CronJobsAdminService.ts
                           ↓
Model / Repository  packages/models/src/models/AgendaJobsModel.ts
                    packages/models/src/models/CronHistoryModel.ts (existing — read-only)
                           ↓
Scheduler Proxy     packages/cron/src/index.ts (AgendaCronJobs — extended with public API)
                           ↓
MongoDB             rocketchat_cron / cron_history
```

---

## Critical: Extending `AgendaCronJobs` with Public Methods

The `scheduler` property on `AgendaCronJobs` is **`private`** (`packages/cron/src/index.ts:59`). The service layer cannot access `cronJobs.scheduler` directly.

**Required Changes** to `packages/cron/src/index.ts`:

```ts
export class AgendaCronJobs {
  private reservedJobs: ReservedJob[] = [];
  private scheduler: Agenda | undefined;

  // --- EXISTING public methods (unchanged) ---
  public get started(): boolean { return Boolean(this.scheduler); }
  public async start(mongo: Db): Promise<void> { /* ... */ }
  public async add(name, schedule, callback) { /* ... */ }
  public async addAtTimestamp(name, when, callback) { /* ... */ }
  public async remove(name) { /* ... */ }
  public async has(jobName) { /* ... */ }

  // --- NEW public methods for admin API ---

  /**
   * Check if a job handler is registered in this process's in-memory definitions.
   * This is needed to guard against triggering jobs that exist in MongoDB
   * but have no handler in the current process (e.g., orphaned calendar docs).
   */
  public hasDefinition(name: string): boolean {
    if (!this.scheduler) return false;
    return Boolean(this.scheduler.getDefinition(name));
  }

  /**
   * Trigger a one-off execution of a named job.
   * Uses scheduler.now() which inserts a new 'normal'-type document.
   * For 'single'-type jobs, callers should use updateNextRunAt() instead.
   */
  public async triggerNow(name: string, data: Record<string, any> = {}): Promise<void> {
    if (!this.scheduler) {
      throw new Error('Scheduler not started');
    }
    await this.scheduler.now(name, data);
  }

  /**
   * Returns the names of all registered job definitions in this process.
   * Used by the admin UI to show which jobs are "active" vs "orphaned".
   */
  public getRegisteredJobNames(): string[] {
    if (!this.scheduler) return [];
    return Object.keys(this.scheduler.getDefinitions());
  }
}
```

These additions are minimal surface — 3 methods, each delegating to the private `scheduler`. This preserves encapsulation while enabling admin operations.

---

## Type Definitions

### `IAgendaJob` — New type in `packages/core-typings/src/IAgendaJob.ts`

```ts
import type { ObjectId } from 'mongodb';

/**
 * Mirrors the document shape stored in the 'rocketchat_cron' collection by Agenda.
 * Derived from the IJob interface in packages/agenda/src/definition/IJob.ts.
 *
 * NOTE: _id is ObjectId (BSON), not string. Agenda creates documents
 * with Mongo-native ObjectId. BaseRaw.findOneById() handles string→ObjectId
 * coercion internally.
 */
export interface IAgendaJob {
  _id: ObjectId;
  name: string;
  type: 'once' | 'single' | 'normal';
  repeatInterval?: string | number;
  repeatTimezone?: string | null;
  repeatAt?: string;
  nextRunAt?: Date | null;
  lastRunAt?: Date;
  lastFinishedAt?: Date;
  lockedAt?: Date | null;
  disabled?: boolean;
  priority?: number;
  failReason?: string;
  failCount?: number;
  failedAt?: Date;
  lastModifiedBy?: string;
  data?: Record<string, any>;
}

export type AgendaJobStatus = 'scheduled' | 'running' | 'failed' | 'disabled' | 'stuck';
```

### `JobSummary` — Aggregation result type

```ts
// packages/core-typings/src/IAgendaJob.ts (continued)
export interface JobSummary {
  _id: string;            // job name (the $group _id)
  count: number;          // number of documents with this name
  nextRunAt: Date | null;
  lastRunAt: Date | null;
  lastFinishedAt: Date | null;
  totalFailCount: number;
  isDisabled: number;     // 0 or 1
  hasLocked: number;      // 0 or 1
}
```

---

## Model Layer

### New: `AgendaJobsRaw`

**File**: `packages/models/src/models/AgendaJobsModel.ts`

```ts
import type { IAgendaJob, JobSummary } from '@rocket.chat/core-typings';
import type { IAgendaJobsModel } from '@rocket.chat/model-typings';
import type { Db, IndexDescription, Filter, FindOptions } from 'mongodb';

import { BaseRaw } from './BaseRaw';

export class AgendaJobsRaw extends BaseRaw<IAgendaJob> implements IAgendaJobsModel {
  constructor(db: Db) {
    // 'rocketchat_cron' is the collection name Agenda uses
    super(db, 'rocketchat_cron');
  }

  protected override modelIndexes(): IndexDescription[] {
    return [
      // Supports admin list sorted by name + next run
      { key: { name: 1, nextRunAt: -1 } },
      // Supports status filter: failed jobs sorted by most recent failure
      { key: { failCount: 1, failedAt: -1 }, sparse: true },
      // Supports status filter: disabled jobs
      { key: { disabled: 1, name: 1 } },
    ];
  }

  /**
   * Aggregation for the default admin list view.
   * Returns one row per distinct job name with aggregated stats.
   */
  async findGroupedByName(): Promise<JobSummary[]> {
    return this.col
      .aggregate<JobSummary>([
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
      ])
      .toArray();
  }

  /**
   * Find paginated jobs with optional filter.
   * Uses BaseRaw.findPaginated() which returns { cursor, totalCount }.
   */
  findPaginatedJobs(
    filter: Filter<IAgendaJob> = {},
    options: FindOptions<IAgendaJob> = {},
  ) {
    return this.findPaginated(filter, options);
  }

  async findOneByName(name: string): Promise<IAgendaJob | null> {
    return this.findOne({ name });
  }
}
```

### `IAgendaJobsModel` — Interface in `packages/model-typings/`

```ts
import type { IAgendaJob, JobSummary } from '@rocket.chat/core-typings';
import type { IBaseModel } from './IBaseModel';
import type { Filter, FindOptions } from 'mongodb';

export interface IAgendaJobsModel extends IBaseModel<IAgendaJob> {
  findGroupedByName(): Promise<JobSummary[]>;
  findPaginatedJobs(
    filter?: Filter<IAgendaJob>,
    options?: FindOptions<IAgendaJob>,
  ): ReturnType<IBaseModel<IAgendaJob>['findPaginated']>;
  findOneByName(name: string): Promise<IAgendaJob | null>;
}
```

### Model Registration in `packages/models/src/index.ts`

```ts
// Add alongside existing model exports:
export { AgendaJobsRaw } from './models/AgendaJobsModel';
```

And register the proxy in `apps/meteor/server/models/raw/` (matching the pattern for `CronHistoryRaw`).

### Existing: `CronHistoryRaw` — TTL Index Addition

```ts
// packages/models/src/models/CronHistoryModel.ts
protected override modelIndexes(): IndexDescription[] {
  return [
    { key: { intendedAt: 1, name: 1 }, unique: true },
    // NEW: TTL index — auto-delete records older than 30 days
    // Does NOT conflict with the above compound unique index.
    // MongoDB handles multiple indexes on the same field independently.
    { key: { intendedAt: 1 }, expireAfterSeconds: 60 * 60 * 24 * 30 },
  ];
}
```

---

## Service Layer

**File**: `apps/meteor/server/services/cronJobs/CronJobsAdminService.ts`

The service centralizes all business logic. Routes never directly access models or `cronJobs`.

```ts
import { cronJobs } from '@rocket.chat/cron';
import { AgendaJobs, CronHistory } from '@rocket.chat/models';
import type { IAgendaJob, ICronHistoryItem } from '@rocket.chat/core-typings';
import { Logger } from '@rocket.chat/logger';
import { escapeRegExp } from '@rocket.chat/string-helpers';

const logger = new Logger('CronJobsAdmin');
const LOCK_LIFETIME_MS = 10 * 60 * 1000; // mirrors Agenda default

export type JobFilter = {
  status?: 'scheduled' | 'running' | 'failed' | 'disabled' | 'stuck';
  name?: string;
};

export type CronJobsListResult = {
  jobs: IAgendaJob[];
  total: number;
  count: number;
  offset: number;
};

export type CronJobHistoryResult = {
  history: ICronHistoryItem[];
  total: number;
  count: number;
  offset: number;
};

export class CronJobsAdminService {
  /**
   * List all jobs from rocketchat_cron with optional filtering.
   */
  async listJobs(
    filter: JobFilter,
    options: { count: number; offset: number; sort: Record<string, 1 | -1> },
  ): Promise<CronJobsListResult> {
    const mongoFilter = this.buildMongoFilter(filter);
    const { cursor, totalCount } = AgendaJobs.findPaginatedJobs(mongoFilter, {
      skip: options.offset,
      limit: Math.min(options.count, 100), // hard cap at 100
      sort: options.sort,
    });
    const [jobs, total] = await Promise.all([cursor.toArray(), totalCount]);
    return { jobs, total, count: jobs.length, offset: options.offset };
  }

  /**
   * Get a single job document by its Mongo _id.
   * BaseRaw.findOneById() handles string→ObjectId coercion.
   */
  async getJob(id: string): Promise<IAgendaJob | null> {
    return AgendaJobs.findOneById(id);
  }

  /**
   * Get execution history for a job name from cron_history.
   */
  async getJobHistory(
    name: string,
    options: { count: number; offset: number },
  ): Promise<CronJobHistoryResult> {
    const { cursor, totalCount } = CronHistory.findPaginated(
      { name },
      {
        skip: options.offset,
        limit: Math.min(options.count, 50),
        sort: { intendedAt: -1 },
      },
    );
    const [history, total] = await Promise.all([cursor.toArray(), totalCount]);
    return { history, total, count: history.length, offset: options.offset };
  }

  /**
   * Trigger a job to run now.
   *
   * For 'single'-type jobs (most Rocket.Chat recurring jobs created via
   * scheduler.every()), we advance nextRunAt to now and clear lockedAt.
   * This preserves Agenda's single-document-per-job guarantee.
   *
   * For 'normal'/'once' types, we use cronJobs.triggerNow() which inserts
   * a new one-off document.
   */
  async triggerJob(id: string, actorUserId: string): Promise<void> {
    const job = await AgendaJobs.findOneById(id);
    if (!job) {
      throw new Error('error-job-not-found');
    }

    // Guard: ensure job handler is registered in this process.
    // Uses the NEW public method on AgendaCronJobs (not private scheduler).
    if (!cronJobs.hasDefinition(job.name)) {
      throw new Error('error-job-not-defined');
    }

    if (job.type === 'single') {
      // Safe trigger: update nextRunAt to now, clear any stale lock
      await AgendaJobs.updateOne(
        { _id: job._id },
        { $set: { nextRunAt: new Date(), lockedAt: null } },
      );
    } else {
      // For 'normal'/'once' type, insert a new one-off execution doc
      await cronJobs.triggerNow(job.name, job.data ?? {});
    }

    logger.info({
      msg: 'Admin triggered cron job',
      jobName: job.name,
      jobId: String(job._id),
      actorUserId,
      type: job.type,
    });
  }

  async disableJob(id: string, actorUserId: string): Promise<void> {
    const job = await this.findJobOrThrow(id);
    await AgendaJobs.updateOne({ _id: job._id }, { $set: { disabled: true } });

    logger.info({
      msg: 'Admin disabled cron job',
      jobName: job.name,
      jobId: String(job._id),
      actorUserId,
    });
  }

  async enableJob(id: string, actorUserId: string): Promise<void> {
    const job = await this.findJobOrThrow(id);
    await AgendaJobs.updateOne(
      { _id: job._id },
      { $set: { disabled: false, nextRunAt: new Date() } },
    );

    logger.info({
      msg: 'Admin enabled cron job',
      jobName: job.name,
      jobId: String(job._id),
      actorUserId,
    });
  }

  /**
   * Unlock a stuck job.
   * Server-enforced guard: only unlocks if lockedAt exceeds lockLifetime.
   */
  async unlockJob(id: string, actorUserId: string): Promise<void> {
    const job = await this.findJobOrThrow(id);
    if (!job.lockedAt) {
      throw new Error('error-job-not-locked');
    }
    const lockedMs = Date.now() - new Date(job.lockedAt).getTime();
    if (lockedMs < LOCK_LIFETIME_MS) {
      throw new Error('error-job-lock-not-expired');
    }
    await AgendaJobs.updateOne(
      { _id: job._id },
      { $set: { lockedAt: null } },
    );

    logger.warn({
      msg: 'Admin unlocked stuck cron job',
      jobName: job.name,
      jobId: String(job._id),
      actorUserId,
      lockedAt: job.lockedAt,
      lockedForMs: lockedMs,
    });
  }

  // ---- Private helpers ----

  private async findJobOrThrow(id: string): Promise<IAgendaJob> {
    const job = await AgendaJobs.findOneById(id);
    if (!job) {
      throw new Error('error-job-not-found');
    }
    return job;
  }

  private buildMongoFilter(filter: JobFilter): Record<string, any> {
    const query: Record<string, any> = {};
    const now = new Date();
    const stuckDeadline = new Date(now.getTime() - LOCK_LIFETIME_MS);

    if (filter.name) {
      // escapeRegExp prevents ReDoS via user-supplied regex metacharacters.
      // Pattern from: apps/meteor/app/api/server/v1/moderation.ts line 37
      query.name = { $regex: escapeRegExp(filter.name), $options: 'i' };
    }

    switch (filter.status) {
      case 'disabled':
        query.disabled = true;
        break;
      case 'failed':
        query.failCount = { $gt: 0 };
        query.disabled = { $ne: true };
        break;
      case 'running':
        query.lockedAt = { $ne: null, $gt: stuckDeadline };
        query.disabled = { $ne: true };
        break;
      case 'stuck':
        query.lockedAt = { $ne: null, $lte: stuckDeadline };
        query.disabled = { $ne: true };
        break;
      case 'scheduled':
        query.lockedAt = null;
        query.disabled = { $ne: true };
        break;
    }
    return query;
  }
}

export const cronJobsAdminService = new CronJobsAdminService();
```

---

## Error Handling Strategy

Errors follow the existing Rocket.Chat pattern for REST APIs:

| Error Code | HTTP Status | Situation |
|---|---|---|
| `error-job-not-found` | 404 | No document with given `_id` in `rocketchat_cron` |
| `error-job-not-defined` | 422 | Job name not registered in `cronJobs` (no handler in this process) |
| `error-job-not-locked` | 422 | Unlock requested but `lockedAt === null` |
| `error-job-lock-not-expired` | 422 | Unlock requested but lock < 10 min old (may be running) |

Route-level error translation:

```ts
try {
  await cronJobsAdminService.triggerJob(id, this.userId);
  return API.v1.success();
} catch (error: unknown) {
  if (error instanceof Error) {
    switch (error.message) {
      case 'error-job-not-found':
        return API.v1.notFound();
      case 'error-job-not-defined':
      case 'error-job-not-locked':
      case 'error-job-lock-not-expired':
        return API.v1.failure(error.message);
      default:
        throw error;
    }
  }
  throw error;
}
```

---

## Key Design Decisions

1. **Public methods on `AgendaCronJobs`**: Instead of accessing `cronJobs.scheduler` (private), we add `hasDefinition()`, `triggerNow()`, and `getRegisteredJobNames()`. This keeps the private field private while exposing a minimal admin API.

2. **`_id` is ObjectId**: `IAgendaJob._id` is correctly typed as `ObjectId` (matching what Agenda stores). `BaseRaw.findOneById(stringId)` handles the string→ObjectId coercion internally. We never use `as any` casts.

3. **`actorUserId` on every mutation**: All mutation methods take the actor's user ID for audit logging. This is passed from `this.userId` in the route handler.

4. **`escapeRegExp` on name filter**: Prevents ReDoS attacks on the `$regex` query, following the exact pattern from `moderation.ts`.

5. **`findJobOrThrow` helper**: DRY extraction for the common pattern of "find or 404".
