# 02 — Job Lifecycle Deep Dive

## Overview

Every job in Rocket.Chat goes through the following state machine, managed entirely by the `Agenda` class in `packages/agenda/src/Agenda.ts`:

```
[CREATED] → [SCHEDULED] → [LOCKED] → [RUNNING] → [COMPLETED]
                ↑                           ↓
                └────────────── [FAILED] ──→ (retry via reschedule)
                                   ↓
                             [STUCK/ZOMBIE]  ← lockedAt > lockLifetime
```

---

## Phase 1: Creation and Scheduling

**`cronJobs.add(name, cronExpression, fn)`** calls:

```ts
// packages/cron/src/index.ts line 144–152
public async add(name: string, schedule: string, callback): Promise<void> {
  await this.define(name, callback);            // registers fn in _definitions
  await this.scheduler.every(schedule, name, {}, {});  // creates/updates Mongo doc
}
```

`Agenda.every()` → `_createIntervalJob()` → `job.repeatEvery(interval)` → `job.computeNextRunAt()` → `job.save()`.

For `type: 'single'` jobs (which is what `every()` creates), `saveJob()` issues a `findOneAndUpdate({ name, type: 'single' }, ..., { upsert: true })`. This means the same job name always has **exactly one document** in `rocketchat_cron`. Re-scheduling updates that document in place.

**`computeNextRunAt()`** supports two interval types:
1. **Cron string** (e.g., `'0 */3 * * *'`): Parsed via `CronTime` from the `cron` package. `_getNextDateFrom(lastRun)` computes next execution.
2. **Human interval** (e.g., `'1 minute'`): Parsed via `humanInterval`. `nextRunAt = lastRun + intervalMs`.

---

## Phase 2: Discovery via Poll

`Agenda.start()` sets up a `setInterval` calling `processJobs()` every `processEvery` (configured as `'1 minute'` in `AgendaCronJobs.start()`).

```ts
// packages/agenda/src/Agenda.ts line 611–621
public async start(): Promise<void> {
  await this._ready;
  this._processInterval = setInterval(
    () => this.processJobs(),
    this._processEvery   // 60,000 ms
  );
  process.nextTick(() => this.processJobs());
}
```

`processJobs()` iterates over all entries in `this._definitions` (in-memory registry) and calls `_jobQueueFilling(name)` for each.

---

## Phase 3: Locking

`_findAndLockNextJob(name, definition)` issues the critical atomic Mongo operation:

```ts
const JOB_PROCESS_WHERE_QUERY = {
  $and: [
    { name: jobName, disabled: { $ne: true } },
    {
      $or: [
        { lockedAt: { $eq: null }, nextRunAt: { $lte: this._nextScanAt } },
        { lockedAt: { $lte: lockDeadline } },  // reclaim expired locks
      ],
    },
  ],
};
const JOB_PROCESS_SET_QUERY = { $set: { lockedAt: now } };

const result = await collection.findOneAndUpdate(
  JOB_PROCESS_WHERE_QUERY,
  JOB_PROCESS_SET_QUERY,
  { returnDocument: 'after', sort: this._sort }
);
```

This operation is safe for multi-process deployments: MongoDB's `findOneAndUpdate` is atomic. Only one node in a cluster wins the lock. The losing node gets `null` and skips.

**Lock expiry** (`lockDeadline = now - lockLifetime`): If a job document has `lockedAt <= lockDeadline`, it has exceeded the maximum lock duration (10 minutes by default). Another process can reclaim it. This handles the crash-recovery case.

---

## Phase 4: Execution

`Job.run()` (line 188–260 of `Job.ts`):

```ts
public run(): Promise<Job> {
  return new Promise(async (resolve, reject) => {
    this.attrs.lastRunAt = new Date();
    this.computeNextRunAt();      // compute next occurrence immediately
    await this.save();            // persist lastRunAt + nextRunAt to Mongo

    // ... jobCallback definition
    try {
      this.agenda.emit('start', this);
      await definition.fn(this, jobCallback);  // or await + auto-callback
    } catch (error) {
      await jobCallback(error);
    }
  });
}
```

On completion (success or failure), `jobCallback()` is called:

```ts
const jobCallback = async (err?: Error) => {
  if (err) {
    this.fail(err);            // sets failReason, failCount++, failedAt
  } else {
    this.attrs.lastFinishedAt = new Date();
  }
  this.attrs.lockedAt = null;  // release lock
  await this.save();           // persist result to rocketchat_cron
  this.agenda.emit(err ? 'fail' : 'success', err, this);
  this.agenda.emit('complete', this);
  resolve(this);
};
```

**`Job.fail()`** (line 173–186):
```ts
public fail(reason: Error | string): Job {
  this.attrs.failReason = reason instanceof Error ? reason.message : reason;
  this.attrs.failCount = (this.attrs.failCount || 0) + 1;
  this.attrs.failedAt = new Date();
  this.attrs.lastFinishedAt = new Date();
  return this;
}
```

---

## Phase 5: CronHistory Persistence (Rocket.Chat Extension)

Rocket.Chat wraps every job handler in `runCronJobFunctionAndPersistResult()` (`packages/cron/src/index.ts` lines 9–40):

```ts
const runCronJobFunctionAndPersistResult = async (fn, jobName) => {
  const { insertedId } = await CronHistory.insertOne({
    _id: Random.id(),
    intendedAt: new Date(),
    name: jobName,
    startedAt: new Date(),
  });
  try {
    const result = await fn();
    await CronHistory.updateOne({ _id: insertedId }, {
      $set: { finishedAt: new Date(), result },
    });
    return result;
  } catch (error) {
    await CronHistory.updateOne({ _id: insertedId }, {
      $set: { finishedAt: new Date(), error: error?.stack ?? error },
    });
    throw error;  // re-throws so Agenda also records failure in rocketchat_cron
  }
};
```

Two failure records are written when a job fails:
1. `cron_history` document: `{ finishedAt, error: stack_trace }`
2. `rocketchat_cron` document: `{ failReason: message, failCount: N, failedAt }`

---

## Phase 6: Stuck / Zombie State

A job is "stuck" when:
- `lockedAt !== null`  
- `lockedAt <= (now - lockLifetime)` — lock has expired

This happens when:
- The Node process crashed mid-execution (SIGKILL, OOM)
- A job ran for longer than `lockLifetime` (10 minutes) without calling `job.touch()`

Agenda handles this automatically on the next `_findAndLockNextJob()` poll (via the `lockedAt: { $lte: lockDeadline }` branch in the WHERE query). However, if `lockLifetime` is exceeded AND no other process is polling for the job (e.g., it's a single-process deployment without the job re-defined after restart), the job remains stuck until next server start.

**Admin fix**: The proposed `POST /api/v1/cron-jobs/:id/unlock` endpoint manually sets `lockedAt: null` on the document, allowing the next poll to pick it up.

---

## Concurrency Model

`AgendaCronJobs` configures Agenda with `defaultConcurrency: 1`, meaning at most **1 instance of each job name runs simultaneously**. The `maxConcurrency: 20` (Agenda default) controls total concurrent jobs across all types.

`_shouldLock(name)` enforces per-job concurrency:
```ts
if (jobDefinition.lockLimit && jobDefinition.lockLimit <= jobDefinition.locked) {
  return false;  // skip — another instance is running
}
```

In a multi-instance Rocket.Chat deployment (multiple Meteor processes against one MongoDB), only one process will win the atomic `findOneAndUpdate` lock per job document. The `lastModifiedBy` field records which Agenda instance last touched the document.
