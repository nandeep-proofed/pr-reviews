# PR Review: fix/PP-1870: Hide submission UI for editors viewing a job already accepted by another editor

**PR:** https://github.com/Proofed/B2BWebserver/pull/2299
**Jira:** https://proofed.atlassian.net/browse/PP-1870
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Non-assigned editor opening a stale "View job" link must not see the in-progress submission UI (download / upload edited copy / upload track changes) | Page-level gate in `JobsPage`: `effectiveJobId` is only set when the job has loaded **and** `isJobTakenByAnotherViewer` is false, so `JobSidebar` never mounts for a taken job | ✅ Addressed |
| Show the offer overview **or** an appropriate "this job has been taken" / not-available state | Implements the second alternative: "Job unavailable" warning toast (10 s) + `?jobId=` cleared from the URL, reusing the existing `showJobConflictError` pattern | ✅ Addressed |
| Assigned editor must continue to see the submission UI unchanged | Gate returns false when `proofedUserId === userId`; covered by unit test | ✅ Addressed |
| Jobs still in `Offered` status must remain acceptable by any candidate | No explicit status check; relies on the OMS invariant that `Offered` jobs have `proofedUserId` unset (verified against OMS API Reference Guide rev 37: the accept endpoint requires the job to be unassigned, and the On Hold / In Queue / re-offer transitions all clear `proofedUserId`). Covered by the `proofedUserId === 0` sentinel test | ✅ Addressed (see Issue 1 — PR description claims a status check that does not exist) |
| Deep link to a deleted/unknown job should not silently render nothing | Page-level `onError` mirrors the panel's existing behavior: "This job was not found." error toast + URL clear | ✅ Addressed (bonus, consistent with existing panel behavior) |

**Beyond-Jira changes (scope creep, all benign):**

- `ChangeJobStatusModal/utils.ts`: `"QA"` string literal → `UserRole.QA`. Behavior-identical (`UserRole.QA === "QA"`); both existing callers (`ChangeJobStatusModal/hooks.tsx:51`, `BulkAssignmentModal/hooks.ts:59`) are unaffected.
- `showJobConflictError` parameterised by role with `UserRole.Reviewer` default, preserving wording at the existing call site (`pages/jobs/hooks.ts:81`).

---

## Architecture Analysis

The fix moves the "is this job mine to act on" decision up to the `/jobs` page instead of patching `JobManagement`'s status-driven rendering. Key design points verified against the codebase:

- **No extra network call.** The page-level `useJobQuery(jobId, ...)` uses the exact key the side panel's `useJobSidePanel` already uses (`[JOBS_QUERY_KEY, jobId]`, both passing the router string), so React Query dedupes the fetch.
- **Helper soundness.** `isJobTakenByAnotherUser` checks only `proofedUserId` truthiness + identity, no status. This is sound: per the OMS API reference, the accept endpoint rejects when the job is assigned, and every transition back to an acceptable state (`Offered`, `On Hold`, `In Queue`) clears `proofedUserId`. So `proofedUserId` set ⇒ job genuinely not acceptable, regardless of status. The gate is actually *stronger* than a status check would be.
- **Toast re-fires correctly on repeat clicks.** Global `QueryClient` uses default `staleTime: 0`, so re-opening the same taken job triggers a refetch and `onSuccess` fires again — no missed feedback from cache hits.
- **Role-agnostic gate is safe.** `/jobs` is restricted to `Editor`/`Reviewer` via `withUserProvided`, and `proofedUserId` is the per-job assignee, so reviewers and dual-role users are handled identically.

The gate lives in the query's `onSuccess`/`onError` callbacks (TanStack Query v4 — supported, and the prevailing pattern in this codebase) rather than a `useEffect`, contrary to the PR description (see Issue 1).

---

## Issues Found

### 1. PR description describes a different implementation than what is shipped

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/utils.ts]**

**Function/Class:** isJobTakenByAnotherUser / PR description

**Severity:** medium

**Problem:** The PR description makes four claims that do not match the shipped code: (1) the helper "returns true only when … the status is past `Offered`" — the helper has no status check at all; (2) "8 unit tests … (… Submitted / Assigned / Approved terminal cases)" — only 4 tests shipped, none status-related; (3) "A `notifiedUnavailableJobIdRef` keyed on `jobId` prevents duplicate toasts" — no such ref exists; (4) "the gate `useEffect`" — the gate runs in the query's `onSuccess`, not an effect.

**Impact:** The shipped code is actually correct (the `proofedUserId` invariant covers the `Offered` carve-out, verified against the OMS API docs), but reviewers approving against the description are approving behavior that isn't there, and future maintainers reading the merged PR description will look for a status check and a dedup ref that don't exist. The squash-merge commit will carry this misleading description.

**Fix:** Update the PR description to match the final implementation: helper checks `proofedUserId` only (with a note on the OMS invariant that acceptable jobs are unassigned), 4 helper tests, gate implemented via `onSuccess`/`onError`, no dedup ref (StrictMode dedup is handled by React Query's fetch deduplication and the immediate URL clear).

### 2. Side panel no longer opens instantly — click feedback gap

**[File: apps/creative-portal/components/pages/jobs/index.tsx]**

**Function/Class:** JobsPage / effectiveJobId

**Severity:** low

**Problem:** Previously `isOpen={jobId !== undefined}` opened the sidebar immediately on row click, and the panel showed its own loader while `useJobSidePanel` fetched the job. Now `isOpen={effectiveJobId !== undefined}` keeps the panel fully closed until the page-level job fetch resolves, for **every** job open (not just deep links). On a slow connection a row click gives no visual feedback until the GET returns.

**Impact:** Perceived dead click on every job open; behavior change for the 99% case (own/offered jobs) introduced to fix the deep-link edge case. Transient fetch errors also now surface as "This job was not found" before the panel opens — though this matches the panel's pre-existing `retry: false` + same `onError`, so error semantics are unchanged, only the timing of the closed panel.

**Fix:** Acceptable as-is given the shared query key makes the wait one round trip, but consider opening the sidebar optimistically while the gate query is in flight, e.g. treat "loading" as open and only force-close once the gate trips:

```typescript
const isGateBlocked = isJobTakenByAnotherViewer(targetJob, user.id);
const effectiveJobId = jobId && !isGateBlocked ? jobId : undefined;
// JobSidebar already renders its own loader while the job query resolves
```

(The panel's own `onError` already handles not-found links, so the page-level gate would only need the taken-by-another-user path.)

### 3. Duplicate `onError` observers once the panel is open

**[File: apps/creative-portal/components/pages/jobs/index.tsx]**

**Function/Class:** JobsPage / useJobQuery onError

**Severity:** low

**Problem:** After the gate passes, both the page-level query and `useJobSidePanel`'s query (same key `[JOBS_QUERY_KEY, jobId]`) are mounted, and each registers its own `onError` showing the identical "This job was not found." toast plus `router.replace`. In TanStack Query v4, `onError` fires per observer, so any query failure while the panel is open (e.g. a refetch after `refreshJobsData` invalidation when the job was just deleted) produces two stacked identical toasts.

**Impact:** Cosmetic duplicate toasts in a narrow failure window. `refetchOnWindowFocus` is globally disabled and `retry: false` matches the panel, so the window is small.

**Fix:** Either drop the toast from the page-level `onError` (keep only the URL clear — the panel observer isn't mounted for the deep-link-to-dead-job case, so keep the full handler only if the panel is closed), or use a module-level/toast-id dedup. Simplest: give the toast a fixed `toastId` if the shared Toast atom supports it.

### 4. Accept-conflict path still hardcodes "reviewer" wording

**[File: apps/creative-portal/components/pages/jobs/hooks.ts]**

**Function/Class:** handleConflictOrFallback (line 81)

**Severity:** low

**Problem:** The PR parameterised `showJobConflictError(role)` specifically so the wording matches who took the job, but the most common conflict surface — an editor losing the accept race (409 from `handleConflictOrFallback`) — still calls `showJobConflictError()` with no argument and therefore always says "accepted by another **reviewer**", even for editing jobs.

**Impact:** Pre-existing wording bug left in place; the app now shows role-correct wording on deep links but role-wrong wording on accept races, which is inconsistent within the same page.

**Fix:** Pass the role at the conflict site where job context is available:

```typescript
showJobConflictError(getRoleNameBasedOnJob(jobInfo.jobType));
```

(Requires threading the job into `handleConflictOrFallback` or its callers.) Fine as a follow-up ticket if out of scope.

### 5. "another qa" toast copy

**[File: apps/creative-portal/components/pages/jobs/utils.ts]**

**Function/Class:** showJobConflictError

**Severity:** low

**Problem:** `role.toLowerCase()` turns `UserRole.QA` into "qa", producing "It has already been accepted by another qa." The new unit test locks this copy in.

**Impact:** Cosmetic only, and hard to reach (`/jobs` is Editor/Reviewer-gated, and QA jobs route through admin flows), but it's user-facing copy.

**Fix:** Map roles to display nouns instead of lowercasing the enum:

```typescript
const ROLE_NOUNS: Partial<Record<UserRole, string>> = {
  [UserRole.QA]: "QA"
};
const noun = ROLE_NOUNS[role] ?? role.toLowerCase();
```

### 6. Jobs page imports a util from another page's modal partial

**[File: apps/creative-portal/components/pages/jobs/index.tsx]**

**Function/Class:** import of getRoleNameBasedOnJob

**Severity:** low

**Problem:** `JobsPage` imports `getRoleNameBasedOnJob` from `components/pages/admin-area/orders/partials/ChangeJobStatusModal/utils` — a page partial belonging to a different feature. Per the repo's reuse-first convention, a util consumed by two unrelated features should be promoted out of the partial (e.g. into `api/jobTypes/utils` or a jobs-level util module).

**Impact:** Layering smell; couples the jobs page to an admin modal's internals and makes future refactors of `ChangeJobStatusModal` riskier.

**Fix:** Move `getRoleNameBasedOnJob` to a neutral shared location and update the three import sites. Non-blocking.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

---

## Tests

- ✅ 4 unit tests for `isJobTakenByAnotherUser` (different user, assignee, `proofedUserId === 0` sentinel, missing `userId`) — covers the helper's actual branches
- ✅ 5 unit tests for `isJobTakenByAnotherViewer` (undefined job, unassigned, self, other user, missing user id)
- ✅ 6 tests for `showJobConflictError` role wording (default + all five roles), existing default-message test updated
- ⚠️ PR description claims 8 helper tests including Submitted/Assigned/Approved status cases — those do not exist (see Issue 1); not a coverage gap since the helper has no status logic, but the description should be corrected
- ❌ No test for the page-level gate itself (`JobsPage` `onSuccess`/`onError` → toast + URL clear + panel suppressed). The pieces are unit-tested but the wiring is only covered by the manual staging plan
- ✅ Manual staging verification plan is concrete and covers the five relevant scenarios (one item still unchecked in the PR)
- ⏭️ Full suite run: skipped — user opted out; must pass before merge per CLAUDE.md

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Gate logic is sound; `Offered` carve-out verified against OMS API docs |
| Regression risk | ⚠️ Medium-low — sidebar open is now fetch-gated for all jobs (Issue 2); error/toast semantics otherwise match the panel's existing behavior |
| Tests | ⚠️ Good unit coverage of helpers; no test of the page-level wiring; description overstates coverage |
| Code quality | ✅ Follows existing patterns (shared query key, existing toast util); minor layering smell (Issue 6) |
| Validation suite | ⏭️ Skipped — user opted out |
| Mergeable state | ⚠️ GitHub reports clean, but validation suite was not run |

---

## Recommendation

**Approve with suggestions**

1. **Fix the PR description before squash-merge** (Issue 1) — it describes a status check, a dedup ref, a gate `useEffect`, and 8 tests that don't exist in the shipped code. The squashed commit will carry this text permanently.
2. **Run the full validation suite** (`npx turbo run test / typecheck / lint / build`) before merging — it was skipped in this review, and CLAUDE.md treats any failure as a hard blocker.
3. Consider opening the sidebar optimistically while the gate query resolves (Issue 2) so row clicks keep instant feedback — or confirm the product is happy with the brief closed-panel delay during staging verification.
4. Follow-ups (non-blocking): role-correct wording on the accept-race conflict toast (Issue 4), "qa" → "QA" copy (Issue 5), relocate `getRoleNameBasedOnJob` out of the admin modal partial (Issue 6), and a small page-level test for the gate wiring.
