# 07 — Performance and Scaling

## Data Volume Projections

| Collection | Docs at Baseline | Docs at Scale (Enterprise Calendar) |
|---|---|---|
| `rocketchat_cron` | 6–10 (core recurring jobs) | 500–50,000 (per-user calendar jobs) |
| `cron_history` | ~8,760/year (hourly jobs) | ~500K–1M+/year (with calendar + TTL: max ~50K retained) |

---

## Query Performance Analysis

### Default List: Grouped Aggregation

`findGroupedByName()` uses `$group` by `name`. For 6 core jobs (6 documents), this is sub-millisecond. For 50,000 calendar jobs, the `$group` scans all documents but produces only 7–8 output rows (one per distinct job name).

**Index leverage**: The `{ name: 1, nextRunAt: -1 }` index provides a `name` prefix that MongoDB can use as a partial covering index for the `$group` key. The aggregation framework will use this for efficient key ordering.

**Worst-case timing**: At 50,000 documents (~100 MB), a full `$group` scan takes ~50–100ms on modern hardware (SSD, sufficient RAM for working set). Acceptable for an admin page accessed by 1–3 users.

**Optimization if needed**: Cache the grouped result in-memory with a 30-second TTL (using a simple `Map` + timestamp). Since the admin page polls every 10 seconds, this reduces the aggregation frequency by 3x.

### Paginated Individual Jobs

For drill-down by job name:
```ts
AgendaJobs.findPaginatedJobs({ name: jobName }, { limit: 50, skip: 0, sort: { nextRunAt: -1 } })
```
Hits `{ name: 1, nextRunAt: -1 }` index → prefix equality on `name` → range scan on `nextRunAt` → sub-millisecond.

### History Query

```ts
CronHistory.findPaginated({ name: jobName }, { limit: 20, sort: { intendedAt: -1 } })
```
Hits `{ intendedAt: 1, name: 1 }` compound index (backward scan for `-1` sort on `intendedAt`). With TTL limiting `cron_history` to 30 days, max ~50K docs → fast.

---

## TTL Index Impact

At 30 days TTL:
- `temporaryUploadCleanup` (hourly): 720 docs retained
- `VideoConferences` (every 3h): 240 docs retained
- `calendar-reminders` (dynamic): bounded by calendar usage
- **Max realistic retention**: 10,000–50,000 docs at ~2KB each → 100 MB max

MongoDB's TTL background task runs once per minute. No application-level intervention needed.

---

## UI Performance

### Table Rendering

The grouped list defaults to 6–8 rows (one per job name). No virtualization needed. The `Pagination` component from `@rocket.chat/fuselage` handles page navigation for the rare case where an admin drills into individual calendar jobs (paginated at 50 per page).

### Polling Overhead

10-second `refetchInterval` on the list endpoint:
- Query hits indexed path → sub-millisecond server time
- Response size: ~2–10 KB JSON (6–8 jobs)
- Network: 10 KB × 6 requests/min × ~2 concurrent admin users = ~120 KB/min. Negligible.
- `keepPreviousData` (React Query) prevents table flash between polls
- `refetchIntervalInBackground: false` (React Query default) — polling stops when browser tab loses focus

### Sort Interaction

`useSort()` from `@rocket.chat/ui-client` manages `sortBy` and `sortDirection` state. The sort is sent as a JSON-stringified MongoDB sort spec in the `sort` query parameter. The server applies it directly to the Mongo `find()` — no client-side sorting.

---

## Pagination Strategy

- **List endpoint**: default `count: 20`, hard cap at 100 (server-enforced in `CronJobsAdminService.listJobs()`)
- **History endpoint**: default `count: 20`, hard cap at 50
- **Offset-based** (not cursor) — appropriate for low-ingestion admin data
- `usePagination()` from `@rocket.chat/ui-client` manages `current` (offset) and `itemsPerPage` state
- `Pagination` component from `@rocket.chat/fuselage` renders page controls consistent with all other admin tables
