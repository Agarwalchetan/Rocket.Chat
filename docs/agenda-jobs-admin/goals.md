# Goals

## G1 — Full Observability for Scheduled Jobs

**Gap**: Admins have no UI to see which jobs exist, their status, or execution history. They must connect to MongoDB or scan server logs.

**Deliverable**: Admin page at `/admin/cron-jobs` listing all jobs from `rocketchat_cron` with:
- Derived status (scheduled, running, failed, disabled, stuck) via `deriveJobStatus()` pure function
- Schedule expression (cron string or human-interval)
- Last/next run timestamps
- Cumulative failure count

The **default view** uses a grouped aggregation (`$group` by `name`) so the row count equals the number of distinct job types — not the total document count. This means 6–8 rows for core jobs even when 50,000 calendar documents exist.

**Success Metric**: Admin can see all job types in a paginated table that loads in < 200ms at P95. Table supports column sorting (name, next run, last run, failures) via `useSort()` and text/status filtering. 10-second polling via `useQuery({ refetchInterval: 10_000 })` provides near-realtime updates.

---

## G2 — Failure Visibility and Diagnostics

**Gap**: When a job fails, `failReason` and `failCount` are written to `rocketchat_cron`, and the full error stack trace is written to `cron_history.error` — but neither is surfaced anywhere in the admin UI.

**Deliverable**: Detail drawer (using `ContextualbarDialog` from `@rocket.chat/ui-client` / `@rocket.chat/fuselage`, matching `ModerationConsolePage` pattern) showing:
- Job metadata: name, type, schedule, `lastModifiedBy` (instance identifier)
- Execution history table from `cron_history` with: timestamp, duration (`finishedAt - startedAt`), status (success/error), expandable error stack trace
- Status badge turns red on failure, yellow on stuck

**Error tracing**: The `name` field connects `rocketchat_cron` documents → `cron_history` records → server logs (`Logger('Cron')`), enabling cross-reference without leaving the browser.

**Success Metric**: Admin can identify a failing job within < 30 seconds of opening the page. Error stack trace is visible without MongoDB access. History shows last 30 days (TTL-indexed via `{ intendedAt: 1 }, expireAfterSeconds: 2592000`).

---

## G3 — Administrative Actions

**Gap**: When a job fails or becomes stuck, there is no recovery path without direct MongoDB access or server restart.

**Deliverable**: Four admin actions exposed via REST API (`POST` endpoints with `permissionsRequired: ['manage-cron-jobs']`), wired to UI buttons with `GenericModal` confirmation dialogs and toast notifications on success/error:

| Action | Implementation Detail | Server-Side Guards |
|---|---|---|
| **Trigger Now** | `single`-type: `updateOne({ _id }, { $set: { nextRunAt: now, lockedAt: null } })` — preserves Agenda's one-doc-per-name guarantee. `normal`-type: `cronJobs.triggerNow(name, data)` (NEW public method). | Must have handler registered: `cronJobs.hasDefinition(name)` (NEW public method). Rejects with `error-job-not-defined` if handler not found. |
| **Disable** | `updateOne({ _id }, { $set: { disabled: true } })`. Agenda skips disabled jobs on next poll. | None beyond permission. |
| **Enable** | `updateOne({ _id }, { $set: { disabled: false, nextRunAt: new Date() } })`. Advances `nextRunAt` so scheduler picks it up within 1 minute. | None beyond permission. |
| **Unlock** | `updateOne({ _id }, { $set: { lockedAt: null } })`. Frees zombie-locked jobs for re-execution. | **Server-enforced** (not just UI): rejects with `error-job-lock-not-expired` if `lockedAt` is less than `lockLifetime` (10 min) old — job may be legitimately running. |

**Success Metric**: Admin can recover a stuck job in < 3 clicks. All 4 actions produce structured audit log entries via `Logger('CronJobsAdmin')` with `actorUserId`, `jobName`, `jobId`.

---

## G4 — Permission-Gated Access

**Gap**: No permission for cron job management exists. The admin panel uses a variety of `view-*` and `manage-*` permission pairs for different features (e.g., `view-moderation-console` + moderation mutation endpoints).

**Deliverable**: Two permissions, seeded on startup via the permissions registry:

| Permission | Grants | Default Role | Endpoints |
|---|---|---|---|
| `view-cron-jobs` | Read access: list, detail, history | `admin` | `GET /cron-jobs`, `GET /cron-jobs/:id`, `GET /cron-jobs/:id/history` |
| `manage-cron-jobs` | Write access: trigger, disable, enable, unlock | `admin` | `POST /cron-jobs/:id/trigger`, `/disable`, `/enable`, `/unlock` |

- Sidebar item visibility: `permissionGranted: () => hasPermission('view-cron-jobs')`
- Route guard: `usePermission('view-cron-jobs')` in `CronJobsRoute.tsx` → `NotAuthorizedPage` on denied
- API enforcement: `permissionsRequired` array on every route handler → HTTP 403 on denied

This split allows operations teams to have read-only dashboard access without action capability.

**Success Metric**: Non-admin users cannot see the sidebar item or call any endpoint (HTTP 403). Users with `view-cron-jobs` only can browse but not mutate.

---

## G5 — Production-Safe Data Management

**Gap**: `cron_history` has no TTL index — it grows unboundedly. At scale, `temporaryUploadCleanup` alone produces 8,760 documents/year. Calendar jobs can produce tens of thousands more.

**Deliverable**:
- **TTL index** on `cron_history`: `{ key: { intendedAt: 1 }, expireAfterSeconds: 2592000 }` for automatic 30-day cleanup. This does NOT conflict with the existing `{ intendedAt: 1, name: 1 }` compound unique index — MongoDB handles multiple indexes on the same field independently.
- **3 supplementary indexes** on `rocketchat_cron` via `AgendaJobsRaw.modelIndexes()`:
  - `{ name: 1, nextRunAt: -1 }` — default list sort
  - `{ failCount: 1, failedAt: -1 }` — failed job filter (sparse: avoids indexing zero-failure majority)
  - `{ disabled: 1, name: 1 }` — disabled job filter

**Success Metric**: `cron_history` stays bounded (~50,000 docs max at 30 days). Admin list query uses index-covered paths (verifiable via `explain()`). Index creation on server startup does not block operations.

---

## G6 — Dynamic Job Name Handling

**Gap**: Calendar services schedule jobs like `calendar-status-scheduler` dynamically. The admin name filter must handle pattern matching without exposing a ReDoS vulnerability.

**Deliverable**: Name filter uses `escapeRegExp()` from `@rocket.chat/string-helpers` before constructing a `$regex` MongoDB query. Admin can type partial names (e.g., "calendar") to find all calendar-related jobs. The grouped aggregation naturally collapses per-user calendar docs into a single row per job type.

**Success Metric**: Typing "calendar" in the filter shows `calendar-reminders` and `calendar-status-scheduler` groups. Malicious regex metacharacters are escaped, preventing $regex catastrophic backtracking.

---

## Non-Goals

- **Job creation via UI**: Jobs are defined in code. Creating via API without a handler is meaningless.
- **Job deletion**: Deleting a document without unregistering the handler creates a dangling definition. Intentionally omitted.
- **Prometheus/StatsD export**: Metrics infrastructure is out of scope for 175 hours. All data for future metrics is available from `cron_history` + `rocketchat_cron` (documented in 10-observability).
- **WebSocket real-time updates**: 10-second HTTP polling provides sufficient freshness with simpler architecture. `keepPreviousData` (React Query) prevents table flash.
- **Email/Slack alerting on failure**: Deferred to future work. This project lays the observability foundation.
- **Enterprise-only gating**: All features are Community Edition (CE). No `ee/` directory dependencies.
