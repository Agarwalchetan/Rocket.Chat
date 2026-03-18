# 10 — Observability and Logging

## Existing Infrastructure

The `packages/cron/src/index.ts` already attaches structured event listeners to the Agenda scheduler:

```ts
scheduler.on('start', (job) => logger.debug({ msg: `Job "${job.attrs.name}" starting`, jobId, jobName, nextRunAt }));
scheduler.on('complete', (job) => logger.info({ msg: `Job "${job.attrs.name}" completed`, duration }));
scheduler.on('fail', (err, job) => logger.error({ msg: `Job "${job.attrs.name}" failed`, failCount, failReason }));
scheduler.on('error:database', (err) => logger.error({ msg: 'Database error in cron scheduler', err }));
```

These use `Logger('Cron')` — structured JSON output compatible with log aggregators. They include `jobId`, `jobName`, `nextRunAt`, and timing data.

---

## New Logging: Admin Actions

All admin mutations are logged via `Logger('CronJobsAdmin')`:

| Action | Level | Fields |
|---|---|---|
| Trigger | `info` | `jobName`, `jobId`, `actorUserId`, `type` |
| Disable | `info` | `jobName`, `jobId`, `actorUserId` |
| Enable | `info` | `jobName`, `jobId`, `actorUserId` |
| Unlock | `warn` | `jobName`, `jobId`, `actorUserId`, `lockedAt`, `lockedForMs` |

**Log level rationale**:
- `info` for expected admin actions (trigger, enable, disable)
- `warn` for unlock (unusual operation, flags potential issue)
- `error` reserved for unexpected failures in the service itself

```ts
logger.info({
  msg: 'Admin triggered cron job',
  jobName: job.name,
  jobId: String(job._id),
  actorUserId,
  type: job.type,
});
```

**Convention**: Always use structured logging (object argument), never string interpolation. Consistent with all `@rocket.chat/logger` usage in the codebase.

---

## Error Tracing: Connecting `cron_history` to Server Logs

When a job fails, two records are created:

1. **`rocketchat_cron`**: `failReason` (message only), `failCount`, `failedAt`
2. **`cron_history`**: `error` field contains `error.stack` (full stack trace)

The admin UI surfaces both:
- **Table**: Red "Failed" badge with `failCount` in the failures column
- **Detail drawer**: `cron_history` records with expandable error stack trace

**Correlation**: The `name` field links records across collections. The `intendedAt` timestamp in `cron_history` approximately matches `lastRunAt` in `rocketchat_cron`, enabling cross-reference with server logs (which log the same `jobName` and `startTime`).

---

## Metrics Available (Data, Not Infrastructure)

This project does NOT introduce Prometheus/StatsD/OpenTelemetry — that is out of scope for 175 hours. However, all data needed for future metrics is available:

| Metric | Source | Derivation |
|---|---|---|
| Job execution duration | `cron_history` | `finishedAt - startedAt` |
| Job failure rate | `cron_history` | `count(error != null) / total` per job per period |
| Stuck job count | `rocketchat_cron` | `count(lockedAt != null AND lockedAt < now - lockLifetime)` |
| Execution lag | `cron_history` | `startedAt - intendedAt` (should be < processEvery = 1 min) |
| Job throughput | `cron_history` | `count(*) per hour` per job |

A follow-up project could expose these via `GET /api/v1/cron-jobs/metrics` in Prometheus text format:
```
cron_job_last_run_at{name="NPS"} 1710762060
cron_job_fail_count{name="NPS"} 0
cron_job_is_stuck{name="NPS"} 0
cron_job_execution_duration_seconds{name="NPS"} 3.412
```

---

## Log Configuration

The `Logger('CronJobsAdmin')` and `Logger('Cron')` loggers respect Rocket.Chat's log level settings. Admins can adjust verbosity via:
- **Admin → Logs → Log Level** setting (global)
- Environment variable: `LOG_LEVEL=debug` (per-process)

At `debug` level, every scheduler poll and job evaluation is logged. At `info`, only completions and failures. The admin actions logger always logs at `info` or `warn` — visible at default log levels.

---

## Future Extension Points

All built on data already produced by this project:

1. **Workspace notification on high `failCount`**: Check `failCount >= N` in the `fail` event handler → send admin notification. No schema changes needed.

2. **Stuck job alerting**: A scheduled job (itself an Agenda job) that queries `rocketchat_cron` for `lockedAt <= now - lockLifetime` → emit notification.

3. **Prometheus endpoint**: `GET /api/v1/cron-jobs/metrics` exporting gauges per job name from `rocketchat_cron` + `cron_history`.

4. **Webhook on failure**: In `AgendaCronJobs.start()`, extend the `fail` handler to POST to a configurable webhook URL when `failCount > threshold`.

5. **Dashboard widgets**: Workspace statistics page (`/admin/info`) could embed cron health summary (total jobs, active failures, stuck count).

All of these are additive — they require no changes to the admin page, API, or database schema established in this project.
