# 04 — API Design

## Convention Reference

All endpoints follow the pattern established in `apps/meteor/app/api/server/v1/moderation.ts`:
- `API.v1.addRoute(path, options, handlers)`
- `authRequired: true` always
- `permissionsRequired` array for enforcement
- `validateParams` with AJV-compiled validators from `@rocket.chat/rest-typings`
- Responses: `API.v1.success(payload)` / `API.v1.failure(code)` / `API.v1.notFound()`

**IMPORTANT**: Rocket.Chat uses **AJV** (not Zod) for request validation in `rest-typings`. Validators are JSON Schema objects compiled via `ajv.compile()`. This is verified from `packages/rest-typings/src/v1/moderation/ReportHistoryProps.ts`.

---

## Endpoint Type Registry (`rest-typings`)

### File: `packages/rest-typings/src/v1/cron-jobs/cronJobs.ts`

This is **required** for `useEndpoint()` to work on the frontend. Without this file, the hooks have no type resolution.

```ts
import type { IAgendaJob, ICronHistoryItem } from '@rocket.chat/core-typings';
import type { PaginatedResult } from '../../helpers/PaginatedResult';

export type CronJobsEndpoints = {
  '/v1/cron-jobs': {
    GET: (params: CronJobsListParamsGET) => PaginatedResult<{
      jobs: IAgendaJob[];
    }>;
  };
  '/v1/cron-jobs/:id': {
    GET: (params: void) => { job: IAgendaJob };
  };
  '/v1/cron-jobs/:id/history': {
    GET: (params: CronJobHistoryParamsGET) => PaginatedResult<{
      history: ICronHistoryItem[];
    }>;
  };
  '/v1/cron-jobs/:id/trigger': {
    POST: (params: void) => void;
  };
  '/v1/cron-jobs/:id/disable': {
    POST: (params: void) => void;
  };
  '/v1/cron-jobs/:id/enable': {
    POST: (params: void) => void;
  };
  '/v1/cron-jobs/:id/unlock': {
    POST: (params: void) => void;
  };
};
```

### File: `packages/rest-typings/src/v1/cron-jobs/index.ts`

```ts
export * from './cronJobs';
export * from './CronJobsListParams';
export * from './CronJobHistoryParams';
```

### Register in `packages/rest-typings/src/v1/index.ts`

```ts
// Add to existing union of endpoint types:
import type { CronJobsEndpoints } from './cron-jobs';

export type V1Endpoints = /* ... existing types ... */ & CronJobsEndpoints;
```

---

## Request Param Validators (AJV — NOT Zod)

Following the pattern in `packages/rest-typings/src/v1/moderation/ReportHistoryProps.ts`:

### File: `packages/rest-typings/src/v1/cron-jobs/CronJobsListParams.ts`

```ts
import type { PaginatedRequest } from '../../helpers/PaginatedRequest';
import { ajv } from '../Ajv';

type CronJobsListProps = {
  name?: string;
  status?: 'scheduled' | 'running' | 'failed' | 'disabled' | 'stuck';
};

export type CronJobsListParamsGET = PaginatedRequest<CronJobsListProps>;

const cronJobsListParamsSchema = {
  type: 'object',
  properties: {
    name: {
      type: 'string',
      nullable: true,
    },
    status: {
      type: 'string',
      enum: ['scheduled', 'running', 'failed', 'disabled', 'stuck'],
      nullable: true,
    },
    count: {
      type: 'integer',
      nullable: true,
    },
    offset: {
      type: 'integer',
      nullable: true,
    },
    sort: {
      type: 'string',
      nullable: true,
    },
  },
  additionalProperties: false,
};

export const isCronJobsListParams = ajv.compile<CronJobsListParamsGET>(cronJobsListParamsSchema);
```

### File: `packages/rest-typings/src/v1/cron-jobs/CronJobHistoryParams.ts`

```ts
import type { PaginatedRequest } from '../../helpers/PaginatedRequest';
import { ajv } from '../Ajv';

export type CronJobHistoryParamsGET = PaginatedRequest;

const cronJobHistoryParamsSchema = {
  type: 'object',
  properties: {
    count: {
      type: 'integer',
      nullable: true,
    },
    offset: {
      type: 'integer',
      nullable: true,
    },
    sort: {
      type: 'string',
      nullable: true,
    },
  },
  additionalProperties: false,
};

export const isCronJobHistoryParams = ajv.compile<CronJobHistoryParamsGET>(cronJobHistoryParamsSchema);
```

---

## Endpoint Specifications

### `GET /api/v1/cron-jobs`

**Purpose**: Paginated list of jobs from `rocketchat_cron`.

**Permissions**: `view-cron-jobs` (read-only permission)

**Query Params**:

| Param | Type | Default | Description |
|---|---|---|---|
| `count` | integer | 20 | Page size (server caps at 100) |
| `offset` | integer | 0 | Skip |
| `sort` | JSON string | `{"name": 1}` | MongoDB sort spec |
| `name` | string | — | Name substring filter (escaped regex) |
| `status` | enum | — | Filter by computed status |

**Response 200**:
```json
{
  "jobs": [IAgendaJob, ...],
  "count": 20,
  "offset": 0,
  "total": 7,
  "success": true
}
```

**Implementation**:
```ts
API.v1.addRoute(
  'cron-jobs',
  {
    authRequired: true,
    permissionsRequired: ['view-cron-jobs'],
    validateParams: isCronJobsListParams,
  },
  {
    async get() {
      const { count, offset, sort, name, status } = this.queryParams;
      const { count: pCount, offset: pOffset } = await getPaginationItems(this);
      const sortObj = sort ? JSON.parse(sort) : { name: 1 };

      const result = await cronJobsAdminService.listJobs(
        { name, status },
        { count: pCount, offset: pOffset, sort: sortObj },
      );
      return API.v1.success(result);
    },
  },
);
```

---

### `GET /api/v1/cron-jobs/:id`

**Purpose**: Single job document from `rocketchat_cron`.

**Permissions**: `view-cron-jobs`

**Path Param**: `id` — the string representation of the Mongo `ObjectId`. `BaseRaw.findOneById()` handles coercion.

**Response 200**: `{ job: IAgendaJob, success: true }`

**Response 404**: `{ success: false, error: 'error-job-not-found' }`

---

### `GET /api/v1/cron-jobs/:id/history`

**Purpose**: Execution history from `cron_history`, looked up by job name.

**Permissions**: `view-cron-jobs`

**Flow**: Route fetches the job by `_id` → extracts `job.name` → queries `cron_history` by name. If the job document is not found, returns 404.

**Query Params**: `count` (default 20, capped 50), `offset`

**Response 200**:
```json
{
  "history": [ICronHistoryItem, ...],
  "count": 20,
  "offset": 0,
  "total": 150,
  "success": true
}
```

---

### `POST /api/v1/cron-jobs/:id/trigger`

**Purpose**: Immediately trigger a job to run.

**Permissions**: `manage-cron-jobs`

**Body**: _(none required)_

**Response 200**: `{ success: true }`

**Error Responses**:
- **404**: `error-job-not-found` — no document with this `_id`
- **422**: `error-job-not-defined` — job name has no handler registered in this process

---

### `POST /api/v1/cron-jobs/:id/disable`

**Purpose**: Set `disabled: true`. Agenda skips disabled jobs on next poll.

**Permissions**: `manage-cron-jobs`

**Response 200**: `{ success: true }`

---

### `POST /api/v1/cron-jobs/:id/enable`

**Purpose**: Set `disabled: false` and advance `nextRunAt` to now.

**Permissions**: `manage-cron-jobs`

**Response 200**: `{ success: true }`

---

### `POST /api/v1/cron-jobs/:id/unlock`

**Purpose**: Clear `lockedAt: null` on a stuck job.

**Permissions**: `manage-cron-jobs`

**Guards**: Server-enforced — rejects if `lockedAt` is within lockLifetime (10 min).

**Response 200**: `{ success: true }`

**Error Responses**:
- **404**: `error-job-not-found`
- **422**: `error-job-not-locked` — job isn't locked
- **422**: `error-job-lock-not-expired` — lock is fresh (may be legitimately running)

---

## Permission Split

Two permissions, matching the Rocket.Chat convention of `view-*` vs `manage-*` (e.g., `view-moderation-console` vs moderation mutation endpoints):

| Permission | Grants | Default Role |
|---|---|---|
| `view-cron-jobs` | `GET /cron-jobs`, `GET /cron-jobs/:id`, `GET /cron-jobs/:id/history` | `admin` |
| `manage-cron-jobs` | `POST /cron-jobs/:id/trigger`, `disable`, `enable`, `unlock` | `admin` |

This split allows operations teams to have read-only observability without action capability.

---

## Route File Structure

```ts
// apps/meteor/app/api/server/v1/cron-jobs.ts
import { cronJobsAdminService } from '../../../../server/services/cronJobs/CronJobsAdminService';
import { isCronJobsListParams, isCronJobHistoryParams } from '@rocket.chat/rest-typings';
import { API } from '../api';
import { getPaginationItems } from '../helpers/getPaginationItems';

// 3 read endpoints — permissionsRequired: ['view-cron-jobs']
// GET /api/v1/cron-jobs
// GET /api/v1/cron-jobs/:id
// GET /api/v1/cron-jobs/:id/history

// 4 mutation endpoints — permissionsRequired: ['manage-cron-jobs']
// POST /api/v1/cron-jobs/:id/trigger
// POST /api/v1/cron-jobs/:id/disable
// POST /api/v1/cron-jobs/:id/enable
// POST /api/v1/cron-jobs/:id/unlock
```

The file is auto-loaded on server start via the existing `v1/` directory scan.
