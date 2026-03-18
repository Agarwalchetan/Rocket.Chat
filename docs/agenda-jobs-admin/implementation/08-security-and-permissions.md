# 08 — Security and Permissions

## Permission Split: `view-cron-jobs` + `manage-cron-jobs`

Following the `view-*` / `manage-*` split established in moderation (`view-moderation-console` for read, elevated permissions for mutations):

| Permission | Grants | Default Role |
|---|---|---|
| `view-cron-jobs` | List, detail, history (`GET` endpoints) + sidebar visibility | `admin` |
| `manage-cron-jobs` | Trigger, disable, enable, unlock (`POST` endpoints) | `admin` |

### Permission Seeding

Added to the permissions seed list (alongside existing permissions like `view-statistics`, `manage-email-inbox`):

```ts
{ _id: 'view-cron-jobs', roles: ['admin'] },
{ _id: 'manage-cron-jobs', roles: ['admin'] },
```

---

## API-Level Enforcement

Every endpoint declares `permissionsRequired`:

```ts
// Read endpoints
API.v1.addRoute('cron-jobs', {
  authRequired: true,
  permissionsRequired: ['view-cron-jobs'],
  validateParams: isCronJobsListParams,
  // ...
});

// Mutation endpoints
API.v1.addRoute('cron-jobs/:id/trigger', {
  authRequired: true,
  permissionsRequired: ['manage-cron-jobs'],
  // ...
});
```

A user without the permission gets:
```json
HTTP/1.1 403 Forbidden
{
  "success": false,
  "error": "User does not have the permissions necessary to perform this action [error-unauthorized]"
}
```

The client-side `usePermission('view-cron-jobs')` check in `CronJobsRoute.tsx` is a UX guard only — the server enforces independently.

---

## Mutation Guards (Beyond Permissions)

### Trigger Guard: Definition Check

```ts
if (!cronJobs.hasDefinition(job.name)) {
  throw new Error('error-job-not-defined');
}
```

Uses the new `public hasDefinition()` method on `AgendaCronJobs` (NOT the private `scheduler`). This prevents triggering jobs that exist in MongoDB but have no handler in the current process — which would create orphaned execution attempts.

### Unlock Guard: Lock Expiry Check

```ts
const lockedMs = Date.now() - new Date(job.lockedAt).getTime();
if (lockedMs < LOCK_LIFETIME_MS) {
  throw new Error('error-job-lock-not-expired');
}
```

Server-enforced guard, not just a UI hint. Prevents unlocking a legitimately running job.

### Enable Guard: NextRunAt Reset

When enabling a disabled job, `nextRunAt` is set to `new Date()` so the job picks up on the next scheduler poll (within 1 minute). Without this, a disabled job with `nextRunAt` in the past would not be picked up until the scheduler recalculates its schedule.

---

## Input Sanitization

### Name Filter: ReDoS Prevention

The `name` query parameter is used as a MongoDB regex:
```ts
import { escapeRegExp } from '@rocket.chat/string-helpers';
query.name = { $regex: escapeRegExp(filter.name), $options: 'i' };
```

`escapeRegExp` escapes all regex metacharacters (`.`, `*`, `+`, `(`, `)`, `[`, `]`, `{`, `}`, `^`, `$`, `|`, `\`, `?`), preventing catastrophic backtracking. This matches the pattern in `apps/meteor/app/api/server/v1/moderation.ts`.

### Job ID Path Parameter

The `:id` path parameter is a string representation of a MongoDB ObjectId. `BaseRaw.findOneById(id)` handles the coercion and returns `null` for malformed IDs. The service layer throws `error-job-not-found` on null, which the route translates to HTTP 404. No special validation needed.

---

## Authentication & CSRF

- **Authentication**: `authRequired: true` on all routes. Enforced via `X-Auth-Token` + `X-User-Id` headers.
- **CSRF**: The header-based auth scheme is inherently CSRF-resistant — browsers don't auto-attach custom headers on cross-origin requests.
- **No cookies**: Rocket.Chat REST API uses header tokens, not cookies. No `SameSite` concern.

---

## Audit Logging

All mutation endpoints log via `Logger('CronJobsAdmin')` with structured fields:

```ts
logger.info({
  msg: 'Admin triggered cron job',
  jobName: job.name,
  jobId: String(job._id),
  actorUserId,  // this.userId from route context
  type: job.type,
});
```

For the unlock action (higher risk), log level is `warn`:
```ts
logger.warn({
  msg: 'Admin unlocked stuck cron job',
  jobName: job.name,
  jobId: String(job._id),
  actorUserId,
  lockedAt: job.lockedAt,
  lockedForMs: lockedMs,
});
```

---

## Rate Limiting

Rocket.Chat's `API.v1` framework has built-in rate limiting configured per-route. For mutation endpoints, the default rate limit is typically sufficient (60 requests per minute per user). If needed, explicit tighter limits can be set:

```ts
API.v1.addRoute('cron-jobs/:id/trigger', {
  authRequired: true,
  permissionsRequired: ['manage-cron-jobs'],
  rateLimiterOptions: { numRequestsAllowed: 10, intervalTimeInMS: 60000 },
  // ...
});
```

This prevents an admin from flooding the system with trigger requests.

---

## Scope: No Delete, No Create

- **No delete endpoint**: Deleting a job document without unregistering the handler from `cronJobs._definitions` creates a dangling definition. Intentionally omitted.
- **No create endpoint**: Jobs are code-defined. Creating via API without a handler is meaningless.
