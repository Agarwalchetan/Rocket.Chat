# Architecture Decisions

## AD1 — REST API Layer (Not DDP/Meteor Method)

**Decision**: Expose all admin endpoints via `API.v1.addRoute()`.

**Rationale**: All recent admin features (moderation, device management, email inbox) use REST. The DDP layer is legacy for admin tooling. REST supports external integrations (monitoring scripts, CI pipelines) and is testable with standard HTTP tools.

**Rejected**: Meteor Methods — tighter coupling, no load balancer caching, harder to test.

---

## AD2 — New `AgendaJobsRaw` Model (Not Reusing Agenda Internals)

**Decision**: Create `AgendaJobsRaw extends BaseRaw<IAgendaJob>` pointing at `rocketchat_cron`, rather than querying through `cronJobs.scheduler._collection` (which is also private).

**Rationale**: Rocket.Chat's model layer (`BaseRaw`) provides `findPaginated()`, `findOneById()` with automatic ObjectId coercion, and `modelIndexes()` for admin indexes — all of which the Agenda class doesn't expose. A separate model keeps admin concerns separate from scheduler concerns.

**Risk acknowledged**: Two model classes pointing at the same collection. Our model uses `override modelIndexes()` for supplementary indexes only — Agenda creates its own `findAndLockNextJobIndex` during `dbInit()`. No conflict.

---

## AD3 — Supplementary Indexes via Model Layer

**Decision**: Add 3 indexes through `AgendaJobsRaw.modelIndexes()`:
- `{ name: 1, nextRunAt: -1 }` — default list sort
- `{ failCount: 1, failedAt: -1 }` — failure filter (sparse)
- `{ disabled: 1, name: 1 }` — disabled filter

**Rationale**: Agenda's existing `findAndLockNextJobIndex` covers scheduler operations but not admin queries (like "all failed jobs sorted by most recent failure"). Adding indexes via the model layer is the standard Rocket.Chat pattern — `CronHistoryRaw`, `RoomsRaw`, etc. all do this.

**Previously incorrect**: The original design used `{ failedAt: -1 }` (non-sparse). Since most jobs have `failedAt: undefined`, a sparse index on `{ failCount: 1, failedAt: -1 }` is more efficient.

---

## AD4 — HTTP Polling (10s) Over WebSocket Real-time

**Decision**: Use `useQuery` with `refetchInterval: 10_000` and `keepPreviousData` (from `@tanstack/react-query`).

**Rationale**: The admin cron jobs page is visited infrequently (typically by 1–3 users). WebSocket real-time would require:
1. A new subscription definition in the DDP layer
2. Server-side change tracking on `rocketchat_cron` (which Agenda writes to directly, not through Rocket.Chat's change stream)
3. Integration testing of the subscription lifecycle

10-second polling is simpler, has no infrastructure cost, and `keepPreviousData` prevents table flash. This is the same approach used in `ModerationConsoleTable.tsx` (which uses `placeholderData: keepPreviousData`).

---

## AD5 — `cron_history` for Execution Logs (Not Embedded in `rocketchat_cron`)

**Decision**: Read execution history from the existing `cron_history` collection via `CronHistory.findPaginated()`.

**Rationale**: Agenda's `rocketchat_cron` documents store only the *latest* failure (`failReason`, `failCount`). Full execution history (start time, finish time, error stack, return value) is already written to `cron_history` by `runCronJobFunctionAndPersistResult()` in `packages/cron/src/index.ts:9–40`. Embedding history within the job document would:
1. Grow documents beyond the 16MB BSON limit for long-running jobs
2. Duplicate data already persisted in `cron_history`
3. Require modifying Agenda's write path

---

## AD6 — Trigger Strategy: `nextRunAt` Update for Single-Type Jobs

**Decision**: For `type: 'single'` jobs (most Rocket.Chat recurring jobs), triggering is done via `updateOne({ _id }, { $set: { nextRunAt: now, lockedAt: null } })`.

**Rejected**: `cronJobs.triggerNow(name, data)` (which calls `scheduler.now()`) — this creates a **new document** for the job. For `single`-type jobs, Agenda guarantees one document per name via `findOneAndUpdate({ name, type: 'single' }, ..., { upsert: true })` in `saveJob()`. Calling `scheduler.now()` bypasses this, creating a second document that can run concurrently with the scheduled one. This is the most dangerous footgun in the entire project.

For `type: 'normal'` or `type: 'once'` jobs, `cronJobs.triggerNow()` is safe because these types expect multiple documents.

---

## AD7 — Extending `AgendaCronJobs` with Public Methods

**Decision**: Add 3 new public methods to `AgendaCronJobs` instead of making `scheduler` public or accessing it directly.

**Rationale**: `scheduler` is `private` (`packages/cron/src/index.ts:59`). Directly accessing it from the service layer would require:
- Making it `public` (leaks the entire Agenda API surface)
- Using `(cronJobs as any).scheduler` (unsafe, fails `turbo typecheck`)

Instead, we add:
```ts
public hasDefinition(name: string): boolean    // wraps scheduler.getDefinition()
public async triggerNow(name, data): void      // wraps scheduler.now()
public getRegisteredJobNames(): string[]       // wraps Object.keys(scheduler.getDefinitions())
```

This preserves encapsulation while exposing a minimal admin API. When the scheduler is eventually extracted to a microservice, only these 3 methods need to become IPC calls.

---

## AD8 — AJV Validators (Not Zod) for Request Validation

**Decision**: Use AJV-compiled JSON Schema validators in `packages/rest-typings/`, matching the existing pattern.

**Rationale**: Rocket.Chat's `rest-typings` package uses `ajv.compile()` for parameter validation (verified from `packages/rest-typings/src/v1/moderation/ReportHistoryProps.ts`). The `validateParams` option in `API.v1.addRoute()` expects an AJV-compatible validator function, not a Zod schema. Using Zod would require a shim layer that doesn't exist in the codebase.

---

## AD9 — Permission Split: `view-cron-jobs` + `manage-cron-jobs`

**Decision**: Two permissions, not one.

**Rationale**: Rocket.Chat consistently separates read from write permissions (e.g., `view-moderation-console` for read, mutation endpoints require elevated permissions). Operations teams frequently need observability without action capability. A single `manage-cron-jobs` permission forces all-or-nothing access.

The `view-cron-jobs` permission gates the sidebar item, page route, and all `GET` endpoints. `manage-cron-jobs` gates all `POST` endpoints (trigger, disable, enable, unlock).
