# Timeline

> Dates follow the official GSoC 2026 schedule.

---

## Community Bonding (April 2 – May 1, 2026)

**Deliverables (no merge expected)**:

- [ ] Read all referenced source files: `packages/agenda/`, `packages/cron/`, `packages/models/src/models/CronHistoryModel.ts`, Moderation admin pages (pattern reference)
- [ ] Set up local dev environment; verify `rocketchat_cron` and `cron_history` collections populate on startup
- [ ] Interview mentors on: multi-process concerns, desired grouping behavior, CE vs EE scope
- [ ] Design spec: wireframe of job table, status badge colors, detail drawer, confirmation modals
- [ ] Open tracking issue on GitHub

---

## Week 1–2 (May 2–15): Core Infrastructure + `rest-typings`

**Goal: Type-safe skeleton that compiles end-to-end**

- [ ] Add `IAgendaJob`, `AgendaJobStatus`, `JobSummary` types to `packages/core-typings/src/IAgendaJob.ts`
- [ ] Create `IAgendaJobsModel` interface in `packages/model-typings/`
- [ ] Create `AgendaJobsRaw` model in `packages/models/src/models/AgendaJobsModel.ts`
- [ ] Register model proxy in `apps/meteor/server/models/raw/`
- [ ] Add indexes via `modelIndexes()`: `{ name: 1, nextRunAt: -1 }`, `{ failCount: 1, failedAt: -1 }` (sparse), `{ disabled: 1, name: 1 }`
- [ ] Add TTL index to `CronHistoryRaw`: `{ intendedAt: 1 }, expireAfterSeconds: 2592000`
- [ ] **Create `packages/rest-typings/src/v1/cron-jobs/` directory**: `cronJobs.ts` (endpoint types), `CronJobsListParams.ts` (AJV validator), `CronJobHistoryParams.ts` (AJV validator), `index.ts`
- [ ] Register `CronJobsEndpoints` in `packages/rest-typings/src/v1/index.ts`
- [ ] Add `view-cron-jobs` and `manage-cron-jobs` to permissions seed
- [ ] Add 3 public methods to `AgendaCronJobs` in `packages/cron/src/index.ts`: `hasDefinition()`, `triggerNow()`, `getRegisteredJobNames()`
- [ ] Run `turbo typecheck` — must pass

**Checkpoint**: Types compile, indexes created on server start, permissions seeded, `rest-typings` exports resolve.

---

## Week 3–4 (May 16–29): Backend API — Read Endpoints

**Goal: 3 GET endpoints working and tested**

- [ ] Create `apps/meteor/app/api/server/v1/cron-jobs.ts`
- [ ] Implement `GET /api/v1/cron-jobs` with AJV-validated query params
- [ ] Implement `GET /api/v1/cron-jobs/:id`
- [ ] Implement `GET /api/v1/cron-jobs/:id/history`
- [ ] All 3 endpoints: `permissionsRequired: ['view-cron-jobs']`
- [ ] Integration test: `curl -H 'X-Auth-Token: ...' http://localhost:3000/api/v1/cron-jobs` returns paginated list
- [ ] Integration test: non-admin user gets 403

**Checkpoint**: Read API is functional and tested.

---

## Week 5–6 (May 30 – June 12): Backend API — Mutation Endpoints + Service

**Goal: All 4 mutation endpoints safe and tested**

- [ ] Create `CronJobsAdminService` in `apps/meteor/server/services/cronJobs/CronJobsAdminService.ts`
- [ ] Implement `POST /api/v1/cron-jobs/:id/trigger` (single-type guard, definition check)
- [ ] Implement `POST /api/v1/cron-jobs/:id/disable`
- [ ] Implement `POST /api/v1/cron-jobs/:id/enable`
- [ ] Implement `POST /api/v1/cron-jobs/:id/unlock` (lock expiry guard)
- [ ] All 4 endpoints: `permissionsRequired: ['manage-cron-jobs']`
- [ ] Structured audit logging via `Logger('CronJobsAdmin')` on every mutation
- [ ] Unit tests for `CronJobsAdminService` with mocked models
- [ ] Security review: `escapeRegExp` on name filter, permission enforcement

**Checkpoint**: Mentor can test all 4 actions via Postman.

---

## Week 7–8 (June 13–26): Frontend — Route + Page Shell

**Goal: Admin page visible in sidebar and navigable**

- [ ] Add route declaration in `routes.tsx` with `IRouterPaths` augmentation
- [ ] Register: `registerAdminRoute('/cron-jobs/:context?/:id?', ...)`
- [ ] Add sidebar item in `sidebarItems.ts` with `permissionGranted: () => hasPermission('view-cron-jobs')`
- [ ] Create `CronJobsRoute.tsx` (permission guard), `CronJobsPage.tsx` (shell)
- [ ] Add i18n keys to `packages/i18n/src/locales/en.json`
- [ ] Verify `useEndpoint('GET', '/v1/cron-jobs')` type-checks (requires `rest-typings` from Week 1-2)

**Checkpoint**: `/admin/cron-jobs` renders without crashing; visible in sidebar for admin users.

---

## Week 9–10 (June 27 – July 10): Frontend — Job Table

**Goal: Working, data-synced, paginated job list**

- [ ] Implement `CronJobsTable.tsx` following `ModerationConsoleTable.tsx` pattern:
  - `usePagination()` + `useSort()` from `@rocket.chat/ui-client`
  - `Pagination` from `@rocket.chat/fuselage`
  - `GenericTable*` from `@rocket.chat/ui-client`
  - `GenericNoResults` from local `components/`
  - `keepPreviousData` from `@tanstack/react-query`
  - 10s polling via `refetchInterval`
- [ ] Implement `CronJobsTableRow.tsx`
- [ ] Implement `CronJobStatusBadge.tsx` with derived status
- [ ] Implement `CronJobsFilter.tsx`: status dropdown + name text input
- [ ] Handle loading, empty, and error states

**Checkpoint**: Table renders all core jobs; pagination works; status badges reflect real state.

---

## Week 11–12 (July 11–24): Frontend — Detail Drawer + Actions

**Goal: Full admin workflow operational**

- [ ] Implement `CronJobDetailDrawer.tsx` (job metadata + history)
- [ ] Implement `CronJobHistoryTable.tsx` (paginated history inside drawer)
- [ ] Implement action buttons: Trigger Now, Disable/Enable toggle, Unlock (conditional)
- [ ] Implement confirmation modals using `GenericModal`
- [ ] Wire mutations via `useCronJobMutations.ts` hooks
- [ ] Toast notifications on success/error

**Checkpoint**: Full flow: list → click row → drawer → trigger → see result in history.

---

## Week 13–14 (July 25 – August 7): Testing + Polish

**Goal: Production-ready with CI passing**

- [ ] Playwright E2E: navigate, verify table, trigger job, verify history
- [ ] Unit tests: `CronJobsAdminService` (> 80% coverage)
- [ ] API integration tests: all 7 endpoints + permission denial
- [ ] Accessibility: ARIA labels, keyboard navigation
- [ ] Performance: verify P95 < 200ms with 1000 synthetic `cron_history` records
- [ ] Fix issues from mentor review

**Checkpoint**: All tests pass in CI.

---

## Week 15 (August 8–14): Documentation + Final Submission

- [ ] PR description with screenshots
- [ ] Update CHANGELOG.md
- [ ] Resolve all TODO/FIXME comments
- [ ] Submit GSoC final evaluation
- [ ] Post demo recording

---

## Final Evaluation (August 25, 2026)

Deliverables: 7 REST endpoints, 1 new model, 1 service, 3 public methods on `AgendaCronJobs`, `rest-typings` registration, 2 permissions, TTL index, admin page with 5 UI components, i18n keys, and full test suite.
