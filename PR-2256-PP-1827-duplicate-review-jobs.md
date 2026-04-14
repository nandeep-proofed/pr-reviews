# PR Review: fix/PP-1827: Prevent duplicate Review jobs from double-click and multiple tabs

**PR:** https://github.com/Proofed/B2BWebserver/pull/2256
**Jira:** https://proofed.atlassian.net/browse/PP-1827
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Prevent duplicate jobs from double-click on deadline warning modal confirm button | `useRef`-based guard (`isCreatingJobRef`) in `handleCreateNewJobAndJobTask` prevents re-entry even when closure captures stale state | ✅ Addressed |
| Prevent duplicate jobs from multiple browser tabs | Server-side preflight check in `addJobWithTasks` fetches existing jobs and returns 409 if active duplicate exists | ✅ Addressed |
| Only 1 job of the same type should exist (Canceled/Skipped should not block) | `ACTIVE_JOB_STATUSES` excludes "Skipped" and "Canceled", allowing re-creation | ✅ Addressed |
| Frontend should update UI after conflict | 409 handler invalidates `ORDERS_QUERY_KEY` and `ORDER_JOBS_QUERY_KEY` caches | ✅ Addressed |
| Apply same guard to bulk endpoint (`addNewJobs`) | `addNewJobs.ts` also checks for duplicates before creation | ✅ Addressed |

No scope creep detected — all changes are directly related to the ticket.

---

## Architecture Analysis

The PR implements a defense-in-depth strategy:

1. **Frontend layer**: `useRef` guard prevents the same React component from firing concurrent `mutateAsync` calls. This addresses the primary root cause (double-click on deadline modal confirm button where `useState` captured stale closure).

2. **BFF layer**: Server-side preflight check via `fetchJobsByOrderId` + `ACTIVE_JOB_STATUSES` filter. Returns 409 Conflict for duplicates. This addresses the secondary cause (multiple browser tabs/users).

3. **Error handling**: Frontend gracefully handles 409 by refreshing the cache, causing the job dropdown to re-render without the already-created job type.

The approach is pragmatic. There is an inherent TOCTOU (time-of-check-time-of-use) window between the `fetchJobsByOrderId` check and the `addJob` call on the server. However, the ticket notes that the backend OMS uses `LockOrder`/`UnlockOrder` to serialize concurrent calls, and the frontend ref guard eliminates the primary cause (double-clicks). The remaining risk is minimal.

---

## Issues Found

### 1. Removed error toast for non-409 errors — relies on global handler

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useAddNewJob.ts]**
**Function/Class:** handleCreateNewJobAndJobTask
**Severity:** low
**Problem:** The original catch block called `showDefaultErrorToast(error)` for all errors. The PR removes this and only handles 409. For non-409 errors (e.g., 500, network failures), there is no explicit toast in this catch block.
**Impact:** The PR description correctly states that the global `QueryClient` mutation `onError` handler (`_app.tsx:111`) already calls `showDefaultErrorToast`. However, this relies on an implicit contract — if someone removes or changes the global handler in the future, error feedback could silently disappear. Additionally, `showDefaultErrorToast` in the global handler receives the raw error, while the removed call site was passing it explicitly, so the behavior should be identical.
**Fix:** This is acceptable as-is since the global handler is confirmed at `pages/_app.tsx:110-111`. Just noting the reliance on the implicit global handler. A comment in the catch block explaining "non-409 errors are surfaced by the global QueryClient onError handler" would make this clearer for future developers, but is optional.

### 2. No logging in `addNewJobs.ts` when duplicate is blocked

**[File: apps/creative-portal/api/mixtures/jobs/addNewJobs/addNewJobs.ts]**
**Function/Class:** addNewJobs
**Severity:** low
**Problem:** In `addJobWithTasks.ts`, duplicate blocking logs a warning with `logger.warn("Duplicate job creation blocked", {...})`. In `addNewJobs.ts`, the duplicate is silently pushed to `createdJobs` with status "failed" — no logging.
**Impact:** Debugging production incidents involving the bulk endpoint would lack visibility into why a job creation was skipped. The `addJobWithTasks` endpoint has good observability; the `addNewJobs` endpoint does not.
**Fix:** Add a `logger.warn` call consistent with `addJobWithTasks`:

```typescript
if (duplicateJob) {
  logger.warn("Duplicate job creation blocked", {
    existingJobId: duplicateJob.id,
    existingJobStatus: duplicateJob.status,
    jobType: jobData.jobType,
    orderId
  });

  createdJobs.push({
    orderId,
    jobId: 0,
    status: "failed",
    error: `A ${jobData.jobType} job already exists on this order.`
  });

  return;
}
```

### 3. Repeated `fetchJobsByOrderId` calls in bulk loop

**[File: apps/creative-portal/api/mixtures/jobs/addNewJobs/addNewJobs.ts]**
**Function/Class:** addNewJobs
**Severity:** medium
**Problem:** `fetchJobsByOrderId` is called inside the `jobs.reduce` loop for every job entry. If the bulk request contains multiple jobs for the same `orderId`, this makes redundant API calls to the OMS backend — once per job, even though the order's jobs haven't changed between iterations.
**Impact:** For a bulk request with N jobs on the same order, this makes N identical API calls to the OMS. This adds latency and unnecessary load on the backend. In practice, the bulk endpoint is likely called with a small number of jobs, so the impact is limited but still wasteful.
**Fix:** Cache `fetchJobsByOrderId` results by `orderId` at the top of the reduce loop:

```typescript
const jobsByOrderCache: Record<number, Job[]> = {};

await jobs.reduce(async (promise, bulkJobData, index) => {
  await promise;
  try {
    const { orderId, jobData, ... } = bulkJobData;

    if (!jobsByOrderCache[orderId]) {
      jobsByOrderCache[orderId] = await fetchJobsByOrderId(
        orderId.toString(),
        requesterId
      );
    }

    const existingJobs = jobsByOrderCache[orderId];
    // ... rest of duplicate check
```

Note: the cache should also be updated after a successful job creation to include the newly created job, but since the reduce runs sequentially and each iteration creates a different job type, this is unlikely to cause false negatives in practice.

### 4. Deadline warning modal confirm button lacks disabled/loading state

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useAddNewJob.ts]**
**Function/Class:** handleCreateNewJobAndJobTask / deadline warning modal
**Severity:** medium
**Problem:** The `useRef` guard correctly prevents the second API call, but the modal's confirm button has no visual feedback — it stays enabled and clickable after the first click. The second click is silently swallowed. From the user's perspective, there's no indication that their first click registered or that the job is being created.
**Impact:** Users may think the app is unresponsive and keep clicking, or navigate away assuming nothing happened. While the `useRef` guard prevents the bug, the lack of UX feedback creates a poor user experience. The Jira ticket itself calls out "The button had no disabled state after being clicked" as part of the root cause.
**Fix:** Pass loading state to the deadline modal so the confirm button shows a loading/disabled state while the job is being created. Since `setDeadlineModalProps` currently accepts a static `callback`, the modal API would need to also accept an `isLoading` prop, or the modal component could manage its own loading state by tracking whether the callback promise has resolved:

```typescript
setDeadlineModalProps({
  text: "Setting this date will result in ...",
  callback: () => {
    handleCreateNewJobAndJobTask({
      previousJob,
      upcomingJobs,
      newJobTaskConfig,
      serviceConfigurations
    });
  },
  isLoading: isCreatingJob
});
```

Alternatively, the modal component could disable the confirm button after the first click internally by wrapping the callback and tracking its pending state. This could be a follow-up PR if the modal refactor is out of scope here.

### 5. No test coverage for `addNewJobs.ts` duplicate prevention

**[File: apps/creative-portal/api/mixtures/jobs/addNewJobs/addNewJobs.ts]**
**Function/Class:** addNewJobs
**Severity:** medium
**Problem:** The PR adds duplicate prevention logic to `addNewJobs.ts` but only has tests for `addJobWithTasks.ts` (5 tests) and `useAddNewJob.ts` (5 tests). There are no tests verifying the duplicate check in the bulk endpoint.
**Impact:** The bulk endpoint's duplicate logic is untested. If it regresses, there's no safety net.
**Fix:** Add tests for `addNewJobs.ts` covering at minimum: (1) bulk creation blocked when active duplicate exists, (2) bulk creation allowed when existing job is Canceled/Skipped.

---

## Tests

- ✅ 5 new BFF tests in `addJobWithTasks.test.ts` (happy path, duplicate blocked, Canceled bypass, Skipped bypass, logger warning)
- ✅ 5 new frontend tests in `useAddNewJob.test.ts` (409 cache invalidation, non-409 preserved, no duplicate toast, state reset, double-click ref guard)
- ❌ No tests for `addNewJobs.ts` duplicate prevention
- ✅ Tests verify edge cases (Canceled/Skipped don't block, ref guard prevents concurrent calls)
- ✅ Tests correctly mock axios error structure for 409 detection
- ✅ PR description claims all 711 existing tests pass

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Addresses both root causes (double-click and multi-tab) |
| Regression risk | ✅ Low — additive changes, no existing behavior modified except toast removal (covered by global handler) |
| Tests | ⚠️ Good coverage for primary endpoint, missing for bulk endpoint |
| Code quality | ✅ Clean, well-commented, follows existing patterns |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**
1. **Add logging** in `addNewJobs.ts` when a duplicate is blocked (consistency with `addJobWithTasks.ts`)
2. **Add tests** for `addNewJobs.ts` duplicate prevention logic
3. **Consider caching** `fetchJobsByOrderId` results in the bulk loop to avoid redundant API calls (low priority — unlikely to cause issues in practice)
