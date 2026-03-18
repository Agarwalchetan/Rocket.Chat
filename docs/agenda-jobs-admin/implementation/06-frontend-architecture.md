# 06 — Frontend Architecture

## Directory Structure

Following the exact pattern of `apps/meteor/client/views/admin/moderation/`:

```
apps/meteor/client/views/admin/cronJobs/
├── CronJobsRoute.tsx            ← route guard (permission check)
├── CronJobsPage.tsx             ← page shell with ContextualBar layout
├── CronJobsTable.tsx            ← paginated job table with filters
├── CronJobsTableRow.tsx         ← individual table row component
├── CronJobsFilter.tsx           ← name text input + status dropdown
├── CronJobStatusBadge.tsx       ← status chip component
├── CronJobDetailDrawer.tsx      ← ContextualbarDialog for job details
├── CronJobHistoryTable.tsx      ← execution history table inside drawer
├── CronJobActions.tsx           ← action buttons + confirmation modals
└── hooks/
    ├── useCronJobs.ts           ← useQuery for job list
    ├── useCronJobDetail.ts      ← useQuery for single job
    ├── useCronJobHistory.ts     ← useQuery for execution history
    └── useCronJobMutations.ts   ← useMutation hooks for all actions
```

---

## Route Registration

**`apps/meteor/client/views/admin/routes.tsx`** additions:

```ts
declare module '@rocket.chat/ui-contexts' {
  interface IRouterPaths {
    'admin-cron-jobs': {
      pathname: `/admin/cron-jobs${`/${string}` | ''}${`/${string}` | ''}`;
      pattern: '/admin/cron-jobs/:context?/:id?';
    };
  }
}

registerAdminRoute('/cron-jobs/:context?/:id?', {
  name: 'admin-cron-jobs',
  component: lazy(() => import('./cronJobs/CronJobsRoute')),
});
```

**`apps/meteor/client/views/admin/sidebarItems.ts`** addition:

```ts
{
  href: '/admin/cron-jobs',
  i18nLabel: 'Cron_Jobs',
  icon: 'clock',
  permissionGranted: (): boolean => hasPermission('view-cron-jobs'),
},
```

---

## Import Sources (VERIFIED from ModerationConsolePage.tsx and ModerationConsoleTable.tsx)

| Import | Source | Verified From |
|---|---|---|
| `Page`, `PageHeader`, `PageContent` | `@rocket.chat/ui-client` | `ModerationConsolePage.tsx:2` |
| `Tabs`, `TabsItem` | `@rocket.chat/fuselage` | `ModerationConsolePage.tsx:1` |
| `Pagination` | `@rocket.chat/fuselage` | `ModerationConsoleTable.tsx:2` |
| `ContextualbarHeader`, `ContextualbarTitle` | `@rocket.chat/fuselage` | `ModConsoleReportDetails.tsx:2` |
| `ContextualbarClose`, `ContextualbarDialog` | `@rocket.chat/ui-client` | `ModConsoleReportDetails.tsx:4` |
| `GenericTable`, `GenericTableBody`, `GenericTableHeader`, `GenericTableHeaderCell`, `GenericTableLoadingTable` | `@rocket.chat/ui-client` | `ModerationConsoleTable.tsx:4-12` |
| `usePagination`, `useSort` | `@rocket.chat/ui-client` | `ModerationConsoleTable.tsx:10-11` |
| `useDebouncedValue`, `useMediaQuery`, `useEffectEvent` | `@rocket.chat/fuselage-hooks` | `ModerationConsoleTable.tsx:3` |
| `useEndpoint`, `useRouter`, `useRouteParameter`, `useTranslation`, `useToastMessageDispatch` | `@rocket.chat/ui-contexts` | `ModerationConsolePage.tsx:3` |
| `keepPreviousData`, `useQuery`, `useMutation`, `useQueryClient` | `@tanstack/react-query` | `ModerationConsoleTable.tsx:14` |
| `useTranslation` | `react-i18next` | Also valid (used in `ModerationConsoleTable.tsx:16`) |
| `GenericNoResults` | `../../../components/GenericNoResults` | Local component, relative import |
| `GenericModal` | `../../../components/GenericModal` | Local component, for confirmation dialogs |

> **Note on `useTranslation`**: Both `@rocket.chat/ui-contexts` (returns `t` function directly) and `react-i18next` (returns `{ t }` destructured) are used in the codebase. The page-level components use `@rocket.chat/ui-contexts`, while table components use `react-i18next`. We follow the same split.

---

## Component Implementations

### `CronJobsRoute.tsx`

```tsx
import { usePermission } from '@rocket.chat/ui-contexts';

import NotAuthorizedPage from '../../notFound/NotAuthorizedPage';
import CronJobsPage from './CronJobsPage';

const CronJobsRoute = () => {
  const canViewCronJobs = usePermission('view-cron-jobs');

  if (!canViewCronJobs) {
    return <NotAuthorizedPage />;
  }

  return <CronJobsPage />;
};

export default CronJobsRoute;
```

### `CronJobsPage.tsx` — Page + ContextualBar Layout

Follows the **exact** layout from `ModerationConsolePage.tsx`:
- Outer `<Page flexDirection='row'>` for side-by-side table + drawer
- Inner `<Page>` for table content with header
- `ContextualbarDialog` renders alongside the inner `<Page>` when a job is selected

```tsx
import { Page, PageHeader, PageContent } from '@rocket.chat/ui-client';
import { useTranslation, useRouteParameter, useRouter } from '@rocket.chat/ui-contexts';
import { useCallback } from 'react';

import CronJobsTable from './CronJobsTable';
import CronJobDetailDrawer from './CronJobDetailDrawer';

const CronJobsPage = () => {
  const t = useTranslation();
  const context = useRouteParameter('context');
  const id = useRouteParameter('id');
  const router = useRouter();

  const handleRowClick = useCallback(
    (jobId: string) => {
      router.navigate({
        pattern: '/admin/cron-jobs/:context?/:id?',
        params: { context: 'info', id: jobId },
      });
    },
    [router],
  );

  const handleCloseDrawer = useCallback(() => {
    router.navigate('/admin/cron-jobs', { replace: true });
  }, [router]);

  return (
    <Page flexDirection='row'>
      <Page>
        <PageHeader title={t('Cron_Jobs')} />
        <PageContent>
          <CronJobsTable onClickRow={handleRowClick} />
        </PageContent>
      </Page>
      {context === 'info' && id && (
        <CronJobDetailDrawer jobId={id} onClose={handleCloseDrawer} />
      )}
    </Page>
  );
};

export default CronJobsPage;
```

### `CronJobDetailDrawer.tsx` — ContextualBar Pattern

```tsx
import { ContextualbarHeader, ContextualbarTitle, Box, Divider } from '@rocket.chat/fuselage';
import { ContextualbarClose, ContextualbarDialog } from '@rocket.chat/ui-client';
import { useTranslation, usePermission } from '@rocket.chat/ui-contexts';

import { useCronJobDetail } from './hooks/useCronJobDetail';
import CronJobHistoryTable from './CronJobHistoryTable';
import CronJobActions from './CronJobActions';

type CronJobDetailDrawerProps = {
  jobId: string;
  onClose: () => void;
};

const CronJobDetailDrawer = ({ jobId, onClose }: CronJobDetailDrawerProps) => {
  const t = useTranslation();
  const canManage = usePermission('manage-cron-jobs');
  const { data, isLoading } = useCronJobDetail(jobId);

  return (
    <ContextualbarDialog onClose={onClose}>
      <ContextualbarHeader>
        <ContextualbarTitle>{t('Job_Details')}</ContextualbarTitle>
        <ContextualbarClose onClick={onClose} />
      </ContextualbarHeader>
      {data?.job && (
        <>
          <Box p={16}>
            <Box fontWeight={700}>{data.job.name}</Box>
            <Box color='hint'>
              {t('Type')}: {data.job.type} | {t('Schedule')}: {data.job.repeatInterval ?? data.job.repeatAt ?? 'once'}
            </Box>
            <Box color='hint'>
              {t('Last_Modified_By')}: {data.job.lastModifiedBy ?? 'N/A'}
            </Box>
          </Box>
          {canManage && (
            <>
              <Divider />
              <CronJobActions job={data.job} />
            </>
          )}
          <Divider />
          <Box p={16} fontWeight={700}>{t('Execution_History')}</Box>
          <CronJobHistoryTable jobId={jobId} />
        </>
      )}
    </ContextualbarDialog>
  );
};

export default CronJobDetailDrawer;
```

### `CronJobsTable.tsx` — Full Pagination + Sort + Filter

Modeled directly on `ModerationConsoleTable.tsx`:

```tsx
import { Pagination } from '@rocket.chat/fuselage';
import { useDebouncedValue } from '@rocket.chat/fuselage-hooks';
import {
  GenericTable,
  GenericTableBody,
  GenericTableHeader,
  GenericTableHeaderCell,
  GenericTableLoadingTable,
  usePagination,
  useSort,
} from '@rocket.chat/ui-client';
import { useEndpoint } from '@rocket.chat/ui-contexts';
import { keepPreviousData, useQuery } from '@tanstack/react-query';
import { useMemo, useState } from 'react';
import { useTranslation } from 'react-i18next';

import GenericNoResults from '../../../components/GenericNoResults';
import CronJobsTableRow from './CronJobsTableRow';
import CronJobsFilter from './CronJobsFilter';

type CronJobsTableProps = {
  onClickRow: (id: string) => void;
};

const CronJobsTable = ({ onClickRow }: CronJobsTableProps) => {
  const { t } = useTranslation();
  const [text, setText] = useState('');
  const [statusFilter, setStatusFilter] = useState<string | undefined>();

  const { sortBy, sortDirection, setSort } = useSort<'name' | 'nextRunAt' | 'lastRunAt' | 'failCount'>('name');
  const {
    current,
    itemsPerPage,
    setItemsPerPage: onSetItemsPerPage,
    setCurrent: onSetCurrent,
    ...paginationProps
  } = usePagination();

  const query = useDebouncedValue(
    useMemo(
      () => ({
        ...(text && { name: text }),
        ...(statusFilter && { status: statusFilter }),
        sort: JSON.stringify({ [sortBy]: sortDirection === 'asc' ? 1 : -1 }),
        count: itemsPerPage,
        offset: current,
      }),
      [current, itemsPerPage, sortBy, sortDirection, statusFilter, text],
    ),
    500,
  );

  const getCronJobs = useEndpoint('GET', '/v1/cron-jobs');

  const { data, isLoading, isSuccess } = useQuery({
    queryKey: ['admin', 'cron-jobs', query],
    queryFn: () => getCronJobs(query),
    meta: { apiErrorToastMessage: true },
    placeholderData: keepPreviousData,
    refetchInterval: 10_000,
  });

  const headers = useMemo(
    () => [
      <GenericTableHeaderCell
        key='name'
        direction={sortDirection}
        active={sortBy === 'name'}
        onClick={setSort}
        sort='name'
      >
        {t('Name')}
      </GenericTableHeaderCell>,
      <GenericTableHeaderCell key='schedule'>
        {t('Schedule')}
      </GenericTableHeaderCell>,
      <GenericTableHeaderCell key='status'>
        {t('Status')}
      </GenericTableHeaderCell>,
      <GenericTableHeaderCell
        key='lastRunAt'
        direction={sortDirection}
        active={sortBy === 'lastRunAt'}
        onClick={setSort}
        sort='lastRunAt'
      >
        {t('Last_Run')}
      </GenericTableHeaderCell>,
      <GenericTableHeaderCell
        key='nextRunAt'
        direction={sortDirection}
        active={sortBy === 'nextRunAt'}
        onClick={setSort}
        sort='nextRunAt'
      >
        {t('Next_Run')}
      </GenericTableHeaderCell>,
      <GenericTableHeaderCell
        key='failCount'
        direction={sortDirection}
        active={sortBy === 'failCount'}
        onClick={setSort}
        sort='failCount'
      >
        {t('Failures')}
      </GenericTableHeaderCell>,
    ],
    [sortDirection, sortBy, setSort, t],
  );

  return (
    <>
      <CronJobsFilter text={text} setText={setText} status={statusFilter} setStatus={setStatusFilter} />
      {isLoading && (
        <GenericTable>
          <GenericTableHeader>{headers}</GenericTableHeader>
          <GenericTableBody>
            <GenericTableLoadingTable headerCells={headers.length} />
          </GenericTableBody>
        </GenericTable>
      )}
      {isSuccess && data.jobs.length > 0 && (
        <>
          <GenericTable>
            <GenericTableHeader>{headers}</GenericTableHeader>
            <GenericTableBody>
              {data.jobs.map((job) => (
                <CronJobsTableRow
                  key={String(job._id)}
                  job={job}
                  onClick={() => onClickRow(String(job._id))}
                />
              ))}
            </GenericTableBody>
          </GenericTable>
          <Pagination
            current={current}
            divider
            itemsPerPage={itemsPerPage}
            count={data.total || 0}
            onSetItemsPerPage={onSetItemsPerPage}
            onSetCurrent={onSetCurrent}
            {...paginationProps}
          />
        </>
      )}
      {isSuccess && data.jobs.length === 0 && <GenericNoResults />}
    </>
  );
};

export default CronJobsTable;
```

### `CronJobsTableRow.tsx`

```tsx
import { GenericTableRow, GenericTableCell } from '@rocket.chat/ui-client';
import type { IAgendaJob } from '@rocket.chat/core-typings';

import CronJobStatusBadge from './CronJobStatusBadge';

type CronJobsTableRowProps = {
  job: IAgendaJob;
  onClick: () => void;
};

const CronJobsTableRow = ({ job, onClick }: CronJobsTableRowProps) => (
  <GenericTableRow onClick={onClick} tabIndex={0} role='link' action>
    <GenericTableCell>{job.name}</GenericTableCell>
    <GenericTableCell>
      <code>{job.repeatInterval ?? job.repeatAt ?? 'once'}</code>
    </GenericTableCell>
    <GenericTableCell>
      <CronJobStatusBadge job={job} />
    </GenericTableCell>
    <GenericTableCell>
      {job.lastRunAt ? new Date(job.lastRunAt).toLocaleString() : '—'}
    </GenericTableCell>
    <GenericTableCell>
      {job.nextRunAt ? new Date(job.nextRunAt).toLocaleString() : '—'}
    </GenericTableCell>
    <GenericTableCell>{job.failCount ?? 0}</GenericTableCell>
  </GenericTableRow>
);

export default CronJobsTableRow;
```

### `CronJobStatusBadge.tsx`

```tsx
import { Badge } from '@rocket.chat/fuselage';
import type { IAgendaJob, AgendaJobStatus } from '@rocket.chat/core-typings';

const LOCK_LIFETIME_MS = 10 * 60 * 1000;

const STATUS_VARIANT: Record<AgendaJobStatus, 'primary' | 'danger' | 'warning' | 'secondary' | 'success'> = {
  scheduled: 'primary',
  running: 'success',
  failed: 'danger',
  stuck: 'warning',
  disabled: 'secondary',
};

export function deriveJobStatus(job: Partial<IAgendaJob>): AgendaJobStatus {
  if (job.disabled) return 'disabled';
  if (job.lockedAt) {
    const lockedMs = Date.now() - new Date(job.lockedAt).getTime();
    return lockedMs > LOCK_LIFETIME_MS ? 'stuck' : 'running';
  }
  if ((job.failCount ?? 0) > 0) return 'failed';
  return 'scheduled';
}

const CronJobStatusBadge = ({ job }: { job: IAgendaJob }) => {
  const status = deriveJobStatus(job);
  return (
    <Badge variant={STATUS_VARIANT[status]}>
      {status.charAt(0).toUpperCase() + status.slice(1)}
    </Badge>
  );
};

export default CronJobStatusBadge;
```

### `CronJobActions.tsx` — Mutation Buttons + Confirmation

```tsx
import { Button, ButtonGroup, Box } from '@rocket.chat/fuselage';
import { useTranslation } from '@rocket.chat/ui-contexts';
import { useState } from 'react';

import type { IAgendaJob } from '@rocket.chat/core-typings';
import GenericModal from '../../../components/GenericModal';
import { deriveJobStatus } from './CronJobStatusBadge';
import { useCronJobTrigger, useCronJobDisable, useCronJobEnable, useCronJobUnlock } from './hooks/useCronJobMutations';

type CronJobActionsProps = {
  job: IAgendaJob;
};

const CronJobActions = ({ job }: CronJobActionsProps) => {
  const t = useTranslation();
  const [confirmAction, setConfirmAction] = useState<string | null>(null);
  const status = deriveJobStatus(job);
  const jobId = String(job._id);

  const triggerMutation = useCronJobTrigger(jobId);
  const disableMutation = useCronJobDisable(jobId);
  const enableMutation = useCronJobEnable(jobId);
  const unlockMutation = useCronJobUnlock(jobId);

  const executeAction = () => {
    switch (confirmAction) {
      case 'trigger': triggerMutation.mutate(); break;
      case 'disable': disableMutation.mutate(); break;
      case 'enable': enableMutation.mutate(); break;
      case 'unlock': unlockMutation.mutate(); break;
    }
    setConfirmAction(null);
  };

  return (
    <Box p={16}>
      <ButtonGroup>
        <Button onClick={() => setConfirmAction('trigger')} disabled={status === 'disabled'}>
          {t('Trigger_Now')}
        </Button>
        {status !== 'disabled' ? (
          <Button onClick={() => setConfirmAction('disable')}>{t('Disable_Job')}</Button>
        ) : (
          <Button onClick={() => setConfirmAction('enable')}>{t('Enable_Job')}</Button>
        )}
        {status === 'stuck' && (
          <Button danger onClick={() => setConfirmAction('unlock')}>{t('Unlock_Job')}</Button>
        )}
      </ButtonGroup>
      {confirmAction && (
        <GenericModal
          variant='warning'
          title={t('Are_you_sure')}
          confirmText={t('Yes')}
          cancelText={t('Cancel')}
          onConfirm={executeAction}
          onCancel={() => setConfirmAction(null)}
          onClose={() => setConfirmAction(null)}
        />
      )}
    </Box>
  );
};

export default CronJobActions;
```

---

## Data Fetching Hooks

### `hooks/useCronJobMutations.ts`

```ts
import { useEndpoint, useToastMessageDispatch } from '@rocket.chat/ui-contexts';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useTranslation } from 'react-i18next';

const useJobMutation = (
  method: 'POST',
  path: string,
  jobId: string,
  successKey: string,
) => {
  const endpoint = useEndpoint(method, path as any);
  const queryClient = useQueryClient();
  const dispatchToast = useToastMessageDispatch();
  const { t } = useTranslation();

  return useMutation({
    mutationFn: () => endpoint({ id: jobId } as any),
    onSuccess: () => {
      dispatchToast({ type: 'success', message: t(successKey) });
      void queryClient.invalidateQueries({ queryKey: ['admin', 'cron-jobs'] });
      void queryClient.invalidateQueries({ queryKey: ['admin', 'cron-job', jobId] });
    },
    onError: (error: Error) => {
      dispatchToast({ type: 'error', message: error.message });
    },
  });
};

export const useCronJobTrigger = (jobId: string) =>
  useJobMutation('POST', '/v1/cron-jobs/:id/trigger', jobId, 'Job_triggered_successfully');

export const useCronJobDisable = (jobId: string) =>
  useJobMutation('POST', '/v1/cron-jobs/:id/disable', jobId, 'Job_disabled_successfully');

export const useCronJobEnable = (jobId: string) =>
  useJobMutation('POST', '/v1/cron-jobs/:id/enable', jobId, 'Job_enabled_successfully');

export const useCronJobUnlock = (jobId: string) =>
  useJobMutation('POST', '/v1/cron-jobs/:id/unlock', jobId, 'Job_unlocked_successfully');
```

### `hooks/useCronJobDetail.ts`

```ts
import { useEndpoint } from '@rocket.chat/ui-contexts';
import { useQuery } from '@tanstack/react-query';

export const useCronJobDetail = (jobId: string) => {
  const getJob = useEndpoint('GET', '/v1/cron-jobs/:id');
  return useQuery({
    queryKey: ['admin', 'cron-job', jobId],
    queryFn: () => getJob({ id: jobId }),
    enabled: Boolean(jobId),
    refetchInterval: 10_000,
  });
};
```

### `hooks/useCronJobHistory.ts`

```ts
import { useEndpoint } from '@rocket.chat/ui-contexts';
import { useQuery } from '@tanstack/react-query';

export const useCronJobHistory = (jobId: string, params: { count: number; offset: number }) => {
  const getHistory = useEndpoint('GET', '/v1/cron-jobs/:id/history');
  return useQuery({
    queryKey: ['admin', 'cron-job', jobId, 'history', params],
    queryFn: () => getHistory({ id: jobId, ...params }),
    enabled: Boolean(jobId),
  });
};
```

---

## State Management

| Concern | Mechanism | Source |
|---|---|---|
| Server data (job list) | `useQuery` + `keepPreviousData` | `@tanstack/react-query` |
| Mutations (trigger, etc) | `useMutation` + `invalidateQueries` | `@tanstack/react-query` |
| Pagination | `usePagination()` → `current`, `itemsPerPage`, setters | `@rocket.chat/ui-client` |
| Sort | `useSort()` → `sortBy`, `sortDirection`, `setSort` | `@rocket.chat/ui-client` |
| Text filter debounce | `useDebouncedValue(query, 500)` | `@rocket.chat/fuselage-hooks` |
| Drawer state | `useRouteParameter('context')` + `useRouteParameter('id')` | `@rocket.chat/ui-contexts` |
| Polling | `refetchInterval: 10_000` | React Query (stops on tab blur) |

---

## i18n Keys

**File**: `packages/i18n/src/locales/en.i18n.json`

All keys are added as new entries. Rocket.Chat's i18n system auto-falls back to `en` for locales that don't have a translation. No need to manually update other locale files — the tooling handles it.

```json
{
  "Cron_Jobs": "Cron Jobs",
  "Schedule": "Schedule",
  "Status": "Status",
  "Last_Run": "Last Run",
  "Next_Run": "Next Run",
  "Failures": "Failures",
  "Job_Details": "Job Details",
  "Execution_History": "Execution History",
  "Trigger_Now": "Trigger Now",
  "Disable_Job": "Disable Job",
  "Enable_Job": "Enable Job",
  "Unlock_Job": "Unlock Job",
  "Job_triggered_successfully": "Job triggered successfully",
  "Job_trigger_failed": "Failed to trigger job",
  "Job_disabled_successfully": "Job disabled successfully",
  "Job_enabled_successfully": "Job enabled successfully",
  "Job_unlocked_successfully": "Job unlocked successfully",
  "Last_Modified_By": "Last Modified By"
}
```

---

## Accessibility

- All table rows have `tabIndex={0}` and `role='link'` for keyboard navigation
- `GenericTableHeaderCell` components use `aria-sort` attributes (handled internally)
- `ContextualbarClose` includes accessible close button with ARIA label
- `GenericModal` confirmation dialogs trap focus and support Escape key dismissal
- Status badge text is readable (not icon-only) for screen readers
