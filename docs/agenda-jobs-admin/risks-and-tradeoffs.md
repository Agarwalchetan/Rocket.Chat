# Risks and Trade-offs

## R1 — Race Condition: Admin Trigger + Scheduled Run

**Description**: Admin triggers job X via `POST /api/v1/cron-jobs/:id/trigger` at the exact same time that the scheduler's `processJobs()` poll fires and picks up job X.

**Mechanism**: Agenda's locking is atomic — `findOneAndUpdate({ name, disabled: { $ne: true }, lockedAt: null, nextRunAt: { $lte: scanTime } }, { $set: { lockedAt: now } })`. If two concurrent callers race, only one wins the `findOneAndUpdate`. The other gets `null` and skips.

**Risk Level**: Low for `single`-type jobs using the `nextRunAt: now` trigger strategy. HIGH if `scheduler.now()` is used (creates a new document, bypassing single-type uniqueness).

**Mitigation**: For `single`-type jobs (everything created via `scheduler.every()`), the trigger endpoint does NOT call `scheduler.now()`. Instead: `updateOne({ _id }, { $set: { nextRunAt: new Date(), lockedAt: null } })`. This preserves Agenda's single-execution guarantee.

---

## R2 — MongoDB Performance: High-Volume Calendar Deployments

**Description**: Deployments with aggressive Calendar usage accumulate thousands of `calendar-reminders` and `calendar-status-<id>` jobs. A naive `find({})` on `rocketchat_cron` without filtering scans the entire collection.

**Quantification**: At 10,000 active calendar jobs, each document is ~1–2 KB. Collection size: ~20 MB. Without index: full collection scan. With `{ name: 1, nextRunAt: -1 }` index, the admin list defaults to a grouped aggregation (`$group` by `name`) — producing one row per distinct job type (6–8 rows). Individual job expansion is paginated (max 100).

**Mitigation**:
- Default list uses `findGroupedByName()` aggregation — row count = distinct job names, not total docs
- Individual job view uses `findPaginatedJobs()` with mandatory pagination (hard cap: 100 per page)
- `{ name: 1, nextRunAt: -1 }` index covers both the aggregation `$group` (prefix: `name`) and pagination sort

---

## R3 — Unlock Action: Double Execution of Running Jobs

**Description**: An admin unlocks a job that is genuinely mid-execution (not stuck — actually running). Agenda's next poll will treat the unlocked document as available and pick it up, resulting in duplicate execution.

**Detection Heuristic**:
- `lockedAt !== null && lockedAt <= (now - lockLifetime)` → **Safely stuck** — lock has exceeded the 10-minute lifetime
- `lockedAt !== null && lockedAt > (now - lockLifetime)` → **Possibly running** — lock is fresh, process may be alive

**Mitigation**: The unlock endpoint has a **server-enforced guard** (not just UI):
```ts
const lockedMs = Date.now() - new Date(job.lockedAt).getTime();
if (lockedMs < LOCK_LIFETIME_MS) {
  throw new Error('error-job-lock-not-expired');
}
```
The UI also shows a warning: "This job's lock has expired. Unlocking may cause re-execution if the original process is still alive."

---

## R4 — `cronJobs` Singleton Coupling

**Description**: The `CronJobsAdminService` calls `cronJobs.hasDefinition()` and `cronJobs.triggerNow()` — new public methods on the singleton. This couples the HTTP layer to the in-process scheduler.

**Risk**: In future microservice extraction, the HTTP service won't have an in-process scheduler.

**Mitigation**: `CronJobsAdminService` is the sole consumer of these methods. When the scheduler is extracted, only this service needs to switch from direct method calls to IPC/RPC. The 3 public methods (`hasDefinition`, `triggerNow`, `getRegisteredJobNames`) form a clean interface boundary.

---

## R5 — `rocketchat_cron` Schema Compatibility

**Description**: `IAgendaJob` must match what Agenda writes. If the `@rocket.chat/agenda` package changes its document structure, our type assertion could break silently.

**Mitigation**: `IAgendaJob` is derived from `IJob` in `packages/agenda/src/definition/IJob.ts`. The `_id` field is correctly typed as `ObjectId` (not `string` — Agenda creates documents with native `ObjectId`). `BaseRaw.findOneById()` handles string→ObjectId coercion internally, so route handlers can safely pass string path parameters.

If the Agenda package changes `IJob`, updating `IAgendaJob` is required — this is a compile-time coupling that fails loudly.

---

## R6 — `cron_history` Unbounded Growth

**Description**: `runCronJobFunctionAndPersistResult()` inserts one document per job execution. Without a TTL index, this collection grows indefinitely:
- `temporaryUploadCleanup` (hourly): 8,760 docs/year
- `calendar-reminders` (dynamic): potentially tens of thousands

**Current State**: `CronHistoryRaw` has `{ intendedAt: 1, name: 1 }` as a unique index — but no TTL index.

**Mitigation**: Add TTL index:
```ts
{ key: { intendedAt: 1 }, expireAfterSeconds: 60 * 60 * 24 * 30 } // 30 days
```

This does NOT conflict with the compound unique index — MongoDB supports multiple indexes on the same field independently. The TTL background task runs once per minute.

**Edge case**: When triggering a job via the admin action, the `intendedAt` value must use `new Date()` (current timestamp), NOT the original `nextRunAt` time. Otherwise, the unique index `{ intendedAt: 1, name: 1 }` would reject the insert if the triggered run coincides with the originally-scheduled run at the same millisecond.

---

## R7 — ReDoS via Name Filter

**Description**: The admin API `name` filter is used as a Mongo regex: `{ $regex: filter.name, $options: 'i' }`. User-supplied regex metacharacters (e.g., `(a+)+$`) can cause catastrophic backtracking.

**Mitigation**: Use `escapeRegExp()` from `@rocket.chat/string-helpers` to escape all metacharacters before constructing the regex:
```ts
import { escapeRegExp } from '@rocket.chat/string-helpers';
query.name = { $regex: escapeRegExp(filter.name), $options: 'i' };
```
This matches the pattern in `apps/meteor/app/api/server/v1/moderation.ts`.

---

## R8 — Multi-Instance Behavior

**Description**: In clustered Rocket.Chat deployments, multiple Meteor processes run against one MongoDB. Each creates its own Agenda instance.

**Impact on admin API**:
- `cronJobs.hasDefinition(name)` checks definitions in **this process only**. If jobs are only defined on certain instances (unlikely but possible with calendar jobs), the guard may incorrectly report "not defined" from Instance A while Instance B has the definition.
- Trigger via `nextRunAt: now` update is safe — whichever instance's poll fires next will pick it up atomically.
- `lastModifiedBy` field in `rocketchat_cron` tracks which instance last saved the document.

**Mitigation**: All core Rocket.Chat jobs are defined on every instance. Calendar-specific jobs are defined per-instance but are `type: 'once'` — triggering them via `nextRunAt` update is safe. The admin UI should show `lastModifiedBy` in the detail drawer for debugging.
