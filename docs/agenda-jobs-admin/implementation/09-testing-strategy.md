# 09 — Testing Strategy

## Layer Breakdown

```
Unit Tests          CronJobsAdminService (mocked models + cronJobs public methods)
                    deriveJobStatus() pure function
                    buildMongoFilter() pure function

Integration Tests   REST API endpoints (real Mongo test DB)
                    Permission enforcement (view-cron-jobs vs manage-cron-jobs)
                    AJV validation rejection

UI Tests            Storybook stories (component isolation — 5 status variants)
                    Accessibility testing (ARIA labels, keyboard nav)

E2E Tests           Playwright (full admin workflow with api fixture)
```

---

## Unit Tests: `CronJobsAdminService`

**Tool**: Jest
**Location**: `apps/meteor/server/services/cronJobs/__tests__/CronJobsAdminService.spec.ts`

```ts
import { CronJobsAdminService } from '../CronJobsAdminService';

// Mock the NEW public methods (not private scheduler)
jest.mock('@rocket.chat/models', () => ({
  AgendaJobs: {
    findOneById: jest.fn(),
    updateOne: jest.fn(),
    findPaginatedJobs: jest.fn(),
  },
  CronHistory: {
    findPaginated: jest.fn(),
  },
}));

jest.mock('@rocket.chat/cron', () => ({
  cronJobs: {
    hasDefinition: jest.fn(),
    triggerNow: jest.fn(),
    getRegisteredJobNames: jest.fn(),
  },
}));

const { AgendaJobs } = require('@rocket.chat/models');
const { cronJobs } = require('@rocket.chat/cron');

describe('CronJobsAdminService', () => {
  const service = new CronJobsAdminService();
  beforeEach(() => jest.clearAllMocks());

  describe('triggerJob', () => {
    it('throws error-job-not-found when job does not exist', async () => {
      AgendaJobs.findOneById.mockResolvedValue(null);
      await expect(service.triggerJob('nonexistent', 'admin-uid'))
        .rejects.toThrow('error-job-not-found');
    });

    it('throws error-job-not-defined when handler not registered', async () => {
      AgendaJobs.findOneById.mockResolvedValue({ _id: 'job1', name: 'NPS', type: 'single' });
      cronJobs.hasDefinition.mockReturnValue(false);
      await expect(service.triggerJob('job1', 'admin-uid'))
        .rejects.toThrow('error-job-not-defined');
    });

    it('updates nextRunAt for single-type jobs — does NOT call triggerNow', async () => {
      AgendaJobs.findOneById.mockResolvedValue({ _id: 'job1', name: 'NPS', type: 'single' });
      cronJobs.hasDefinition.mockReturnValue(true);
      AgendaJobs.updateOne.mockResolvedValue({ matchedCount: 1 });

      await service.triggerJob('job1', 'admin-uid');

      expect(AgendaJobs.updateOne).toHaveBeenCalledWith(
        { _id: 'job1' },
        expect.objectContaining({ $set: expect.objectContaining({ lockedAt: null }) }),
      );
      expect(cronJobs.triggerNow).not.toHaveBeenCalled();
    });

    it('calls cronJobs.triggerNow for normal-type jobs', async () => {
      AgendaJobs.findOneById.mockResolvedValue({
        _id: 'job2', name: 'calendar-reminders', type: 'normal', data: { userId: '123' },
      });
      cronJobs.hasDefinition.mockReturnValue(true);

      await service.triggerJob('job2', 'admin-uid');

      expect(cronJobs.triggerNow).toHaveBeenCalledWith('calendar-reminders', { userId: '123' });
    });
  });

  describe('unlockJob', () => {
    const LOCK_LIFETIME_MS = 10 * 60 * 1000;

    it('throws error-job-not-locked when lockedAt is null', async () => {
      AgendaJobs.findOneById.mockResolvedValue({ _id: 'job1', lockedAt: null });
      await expect(service.unlockJob('job1', 'admin-uid'))
        .rejects.toThrow('error-job-not-locked');
    });

    it('throws error-job-lock-not-expired when lock is fresh (< 10 min)', async () => {
      const recentLock = new Date(Date.now() - 60_000);
      AgendaJobs.findOneById.mockResolvedValue({ _id: 'job1', lockedAt: recentLock });
      await expect(service.unlockJob('job1', 'admin-uid'))
        .rejects.toThrow('error-job-lock-not-expired');
    });

    it('unlocks when lock exceeds lockLifetime', async () => {
      const expiredLock = new Date(Date.now() - LOCK_LIFETIME_MS - 1000);
      AgendaJobs.findOneById.mockResolvedValue({ _id: 'job1', name: 'NPS', lockedAt: expiredLock });
      AgendaJobs.updateOne.mockResolvedValue({ matchedCount: 1 });

      await service.unlockJob('job1', 'admin-uid');
      expect(AgendaJobs.updateOne).toHaveBeenCalledWith(
        { _id: 'job1' },
        { $set: { lockedAt: null } },
      );
    });
  });

  describe('buildMongoFilter', () => {
    it('escapes regex metacharacters in name filter', () => {
      const filter = (service as any).buildMongoFilter({ name: 'test(abc)+' });
      expect(filter.name.$regex).toBe('test\\(abc\\)\\+');
    });

    it('returns failCount > 0 for status=failed', () => {
      const filter = (service as any).buildMongoFilter({ status: 'failed' });
      expect(filter.failCount).toEqual({ $gt: 0 });
      expect(filter.disabled).toEqual({ $ne: true });
    });

    it('returns lockedAt + deadline for status=stuck', () => {
      const filter = (service as any).buildMongoFilter({ status: 'stuck' });
      expect(filter.lockedAt.$ne).toBe(null);
      expect(filter.lockedAt.$lte).toBeInstanceOf(Date);
    });
  });
});
```

---

## Unit Tests: `deriveJobStatus`

**Location**: `apps/meteor/client/views/admin/cronJobs/__tests__/deriveJobStatus.spec.ts`

```ts
import { deriveJobStatus } from '../CronJobStatusBadge';

describe('deriveJobStatus', () => {
  const LOCK_LIFETIME = 600_000; // 10 min

  test.each([
    [{ disabled: true }, 'disabled'],
    [{ disabled: true, failCount: 5 }, 'disabled'],           // disabled takes priority
    [{ disabled: true, lockedAt: new Date() }, 'disabled'],   // disabled takes priority over running
    [{ lockedAt: new Date(Date.now() - 1000) }, 'running'],   // fresh lock
    [{ lockedAt: new Date(Date.now() - LOCK_LIFETIME - 1) }, 'stuck'], // expired lock
    [{ failCount: 3 }, 'failed'],
    [{ failCount: 0 }, 'scheduled'],
    [{}, 'scheduled'],
  ])('returns %s status for %o', (job, expected) => {
    expect(deriveJobStatus(job as any)).toBe(expected);
  });
});
```

---

## API Integration Tests

**Location**: `apps/meteor/tests/end-to-end/api/cron-jobs.spec.ts`

Uses the existing test infrastructure: `credentials` (admin token), `request` (supertest), `createUser`/`login`/`deleteUser`.

```ts
import { credentials, request } from '../../data/api-data';
import { createUser, login, deleteUser } from '../../data/users.helper';

describe('Cron Jobs Admin API', () => {
  let regularUser: any;
  let regularCredentials: Record<string, string>;

  before(async () => {
    regularUser = await createUser({ roles: ['user'] });
    regularCredentials = await login(regularUser.username, 'password');
  });

  after(async () => {
    await deleteUser(regularUser);
  });

  describe('GET /api/v1/cron-jobs', () => {
    it('returns 403 for non-admin user', async () => {
      await request
        .get('/api/v1/cron-jobs')
        .set(regularCredentials)
        .expect(403);
    });

    it('returns paginated job list for admin', async () => {
      const { body } = await request
        .get('/api/v1/cron-jobs')
        .set(credentials)
        .expect(200);

      expect(body.success).to.be.true;
      expect(body.jobs).to.be.an('array').that.is.not.empty;
      expect(body.total).to.be.a('number').greaterThan(0);
      expect(body.count).to.be.a('number');
      expect(body.offset).to.equal(0);
    });

    it('filters by name substring', async () => {
      const { body } = await request
        .get('/api/v1/cron-jobs?name=NPS')
        .set(credentials)
        .expect(200);

      body.jobs.forEach((job: any) => {
        expect(job.name.toLowerCase()).to.include('nps');
      });
    });

    it('filters by status=failed', async () => {
      const { body } = await request
        .get('/api/v1/cron-jobs?status=failed')
        .set(credentials)
        .expect(200);

      body.jobs.forEach((job: any) => {
        expect(job.failCount).to.be.greaterThan(0);
      });
    });

    it('rejects invalid status via AJV validation', async () => {
      await request
        .get('/api/v1/cron-jobs?status=invalid')
        .set(credentials)
        .expect(400);
    });

    it('supports pagination params', async () => {
      const { body } = await request
        .get('/api/v1/cron-jobs?count=2&offset=0')
        .set(credentials)
        .expect(200);

      expect(body.jobs.length).to.be.at.most(2);
    });
  });

  describe('POST /api/v1/cron-jobs/:id/trigger', () => {
    it('returns 403 for user with view-cron-jobs only', async () => {
      // Non-admin user doesn't have manage-cron-jobs
      const listRes = await request.get('/api/v1/cron-jobs').set(credentials);
      const jobId = listRes.body.jobs[0]._id;
      await request
        .post(`/api/v1/cron-jobs/${jobId}/trigger`)
        .set(regularCredentials)
        .expect(403);
    });

    it('triggers existing job for admin', async () => {
      const listRes = await request.get('/api/v1/cron-jobs').set(credentials);
      const jobId = listRes.body.jobs[0]._id;
      await request
        .post(`/api/v1/cron-jobs/${jobId}/trigger`)
        .set(credentials)
        .expect(200);
    });
  });

  describe('POST /api/v1/cron-jobs/:id/unlock', () => {
    it('returns error-job-not-locked for an unlocked job', async () => {
      const listRes = await request.get('/api/v1/cron-jobs').set(credentials);
      const job = listRes.body.jobs.find((j: any) => !j.lockedAt);
      if (!job) return; // skip if all jobs are locked

      const { body } = await request
        .post(`/api/v1/cron-jobs/${job._id}/unlock`)
        .set(credentials);

      expect(body.success).to.be.false;
      expect(body.error).to.equal('error-job-not-locked');
    });
  });

  describe('GET /api/v1/cron-jobs/:id/history', () => {
    it('returns execution history', async () => {
      const listRes = await request.get('/api/v1/cron-jobs').set(credentials);
      const jobId = listRes.body.jobs[0]._id;

      const { body } = await request
        .get(`/api/v1/cron-jobs/${jobId}/history?count=5`)
        .set(credentials)
        .expect(200);

      expect(body.history).to.be.an('array');
      expect(body.total).to.be.a('number');
    });
  });
});
```

---

## Playwright E2E Tests

**Location**: `apps/meteor/tests/e2e/admin-cron-jobs.spec.ts`

Uses the Playwright `api` fixture from `apps/meteor/tests/e2e/utils/` for data setup:

```ts
import { test, expect } from './utils/test'; // custom fixture with `api`, `page`
import { createAuxContext } from './utils/AuxContext';

test.describe('Admin Cron Jobs', () => {
  // Use the api fixture for data preconditions
  test.beforeAll(async ({ api }) => {
    // Ensure cron jobs are running (they start automatically on server boot)
    const response = await api.get('/api/v1/cron-jobs');
    expect(response.status()).toBe(200);
  });

  test('Cron Jobs link appears in admin sidebar', async ({ page }) => {
    await page.goto('/admin');
    const sidebarLink = page.getByRole('link', { name: /cron jobs/i });
    await expect(sidebarLink).toBeVisible();
  });

  test('Cron Jobs page shows job table', async ({ page }) => {
    await page.goto('/admin/cron-jobs');
    // Wait for table to load
    await expect(page.getByText('NPS')).toBeVisible({ timeout: 10_000 });
    // Verify pagination exists
    await expect(page.locator('[class*="Pagination"]')).toBeVisible();
  });

  test('Name filter narrows results', async ({ page }) => {
    await page.goto('/admin/cron-jobs');
    const filterInput = page.getByPlaceholderText(/search/i);
    await filterInput.fill('NPS');
    await page.waitForTimeout(1000); // debounce
    const rows = page.locator('tr[role="link"]');
    const count = await rows.count();
    expect(count).toBeGreaterThanOrEqual(1);
    for (let i = 0; i < count; i++) {
      const text = await rows.nth(i).textContent();
      expect(text?.toLowerCase()).toContain('nps');
    }
  });

  test('Clicking a job row opens ContextualBar detail drawer', async ({ page }) => {
    await page.goto('/admin/cron-jobs');
    await page.getByText('NPS').click();
    // ContextualbarDialog should appear
    await expect(page.getByText(/job details/i)).toBeVisible();
    await expect(page.getByText(/execution history/i)).toBeVisible();
    // URL should update with context param
    await expect(page).toHaveURL(/\/admin\/cron-jobs\/info\/.+/);
  });

  test('Close button dismisses drawer', async ({ page }) => {
    await page.goto('/admin/cron-jobs');
    await page.getByText('NPS').click();
    await page.getByRole('button', { name: /close/i }).click();
    await expect(page.getByText(/job details/i)).not.toBeVisible();
    await expect(page).toHaveURL('/admin/cron-jobs');
  });

  test('Non-admin cannot access cron jobs page', async ({ browser }) => {
    const { page: regularPage } = await createAuxContext(browser, { roles: ['user'] });
    await regularPage.goto('/admin/cron-jobs');
    await expect(regularPage.getByText(/not authorized/i)).toBeVisible();
    await regularPage.close();
  });
});
```

**E2E Test Data Dependencies**: Core Rocket.Chat jobs (NPS, statistics, etc.) are registered on every server start via `apps/meteor/server/startup/cron.ts`. The `test.beforeAll` hook verifies the API returns data before running visual assertions. If the test environment runs with a fresh database, jobs will exist after the first `processEvery` (1 minute) poll cycle.

---

## Storybook UI Stories

**Location**: `apps/meteor/client/views/admin/cronJobs/CronJobStatusBadge.stories.tsx`

```tsx
import type { Meta, StoryObj } from '@storybook/react';
import CronJobStatusBadge from './CronJobStatusBadge';

const meta: Meta<typeof CronJobStatusBadge> = {
  title: 'Admin/CronJobs/CronJobStatusBadge',
  component: CronJobStatusBadge,
};

export default meta;
type Story = StoryObj<typeof CronJobStatusBadge>;

export const Scheduled: Story = { args: { job: { name: 'NPS', failCount: 0 } as any } };
export const Running: Story = { args: { job: { name: 'NPS', lockedAt: new Date() } as any } };
export const Failed: Story = { args: { job: { name: 'NPS', failCount: 3 } as any } };
export const Disabled: Story = { args: { job: { name: 'NPS', disabled: true } as any } };
export const Stuck: Story = {
  args: { job: { name: 'NPS', lockedAt: new Date(Date.now() - 700_000) } as any },
};
```

---

## Coverage Targets

| Layer | Target | Rationale |
|---|---|---|
| `CronJobsAdminService` | > 90% line | Core business logic with multiple error paths |
| `deriveJobStatus` | 100% | Pure function, exhaustive `test.each` |
| `buildMongoFilter` | 100% | Pure function, covers each status case + ReDoS escape |
| REST endpoints | 100% of status codes | Permission enforcement + AJV rejection critical |
| UI (Storybook) | All 5 status variants + loading + empty | Visual regression coverage |
| E2E | Full journey: sidebar → table → filter → drawer → close | Smoke test for integration |
