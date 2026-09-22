# PR Review: feature/PP-2095: Offer the next jobs in the panel after a submission

**PR:** https://github.com/Proofed/B2BWebserver/pull/2463
**Jira:** https://proofed.atlassian.net/browse/PP-2095
**Status:** Jira not reachable in this session — the Atlassian MCP server is unauthenticated, so the requirement table below is built from the requirement numbers quoted in the PR body and in the code comments, **not** from the ticket itself. Re-run with Jira authorised before treating the mapping as verified.
**Base:** `feature/PP-2095-latest-job-copy-file-based-orders` (PR #2453) — stacked, review scope is the requirement-6 diff only.
**Reviewed at:** `6a7c42f29` (3 commits, 41 files, +2574 / −128)

---

## What this means for users (non-technical summary)

1. **If someone else grabs a job a split second before you do, the whole "what's next" panel disappears.** You submit a job, the panel offers you three more, you press Accept on one, and another creative has just taken it. Instead of showing you the error and leaving the other two jobs on screen, the panel shuts completely and you are dropped back to the dashboard. This is the exact situation the feature was designed to handle gracefully, so it is worth fixing before release.
2. **A slow, failing refresh after a submission can close a panel you have since opened for a different job.** Submit a job, then click into another job while the page is still catching up. If that catch-up fails, the panel you are now looking at is closed without explanation.
3. **The pay figure on an offered job can sit as a grey placeholder forever** if the information it is calculated from cannot be loaded. There is no message telling you why — it just never resolves. (The main jobs table already behaves this way, so this is consistent rather than new.)
4. **The "open this job" arrow on each card announces the order number, not the job**, which is confusing for anyone using a screen reader.
5. **The wording on the panel still needs a developer and a deploy to change**, which the ticket asked to avoid. The author has flagged this for the PO — there is no content-management system in this repo to hang it on.

---

## Jira Requirements vs Implementation

Requirement numbering taken from the PR description and in-code comments; **not** verified against the ticket (see Status above).

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| Req 6.1 — submitting a job keeps the panel open in a Submitted state instead of closing it | `useSubmittedPanel` owns the state; `hooks.ts` calls `markSubmitted` on the panel's own job; `JobSidebar` renders `SubmittedNextJobs` in place of `JobManagement` | ✅ Addressed |
| Req 6.2 — offer up to three next jobs, assigned first then the queue | `selectNextJobs` concatenates the two pre-sorted lists, de-dupes, caps at `MAX_NEXT_JOBS = 3` | ✅ Addressed |
| Req 6.3 — show fewer when fewer are available | Same function; covered by `utils.test.ts` | ✅ Addressed |
| Req 6.4 — close the panel when nothing is available | Auto-close effect in `useSubmittedPanel`, held until `hasSettledSinceSubmit` | ✅ Addressed |
| Req 6.5 — the user can open an offered job from its card | Chevron `IconButton` → `onOpenJob` → `router.push("?jobId=…")` | ✅ Addressed |
| Req 6.6 — panel copy changeable without a code change | Strings live in `SubmittedNextJobs/consts.ts`; a code change and a deploy are still required. Author has flagged it rather than papered over it | ⚠️ Partial — needs a PO decision |
| Validation rule 10 — unavailable lists fall back to close-on-submit | `refreshJobsData` gains an opt-in `throwOnError` applied only to the two panel-list query keys; the submit path catches and calls `closePanel` | ⚠️ Partial — correct in the intended case, but `closePanel` is unguarded (Issue 2) |
| Req 5.2 / 5.4 — pre-edited indicator on the card | `useNextJobCard` → `hasCompletedPreEdit(orderJobs)` → `AiEditIndicator` | ✅ Addressed |
| Req 3.3 / validation rule 7 — skip failed AI passes when resolving source copy | `findCompletedJob.ts` review fixes (transitive comparator, sequence-aware latest) | ✅ Addressed |
| Q2 (refinement) — assigned jobs read "Start", queue jobs read "Accept" | `getJobActionButtonLabelAndAction` via `getJobActionState`, shared with the table | ✅ Addressed |

**Scope beyond requirement 6.** The PR also carries review fixes against the requirement 1–5 code already in #2453: `getAiEditKind` legacy bare-`"AI"` classification, `hasCompletedPreEdit` passing the order's jobs, `findLatestCompletedJob`'s comparator and sequence handling, the `FilesSection` reviewer-credit fix, and the `resolveFrom` discriminator on `useSourceJobWorkItemVersion`. These are declared in the PR body and are genuine fixes rather than drive-by refactors, but they do widen the blast radius — `FilesSection` is a `packages/shared` export consumed by both portals.

---

## Architecture Analysis

The shape is sound and the reuse discipline is better than average for this repo.

- **One rule for "is this job claimable."** `getJobActionState` (`JobTable/utils.ts:131`) is extracted from the table's `Action` partial and consumed by both the table and the panel's cards. This is the right call — it makes it structurally impossible for a job to be offered on a card and refused in the table. The `Action` partial genuinely got *simpler*, not just re-routed.
- **Pay is derived, not read off the row.** `getDisplayPayAmountForJob` gains `withBonus` so the card can hand the bonus to `Price` separately, exactly as the table's Pay cell does (`consts.tsx:270–278`). I verified the card's arithmetic matches the Pay cell's inline computation including the `isEstimated` rule — the figures will agree.
- **State lives in a hook, `index.tsx` is UI-only.** `SubmittedNextJobs/index.tsx` and `NextJobCard/index.tsx` both hold no `useState`/`useEffect`; `useSubmittedPanel` and `useNextJobCard` carry it. Folder structure, `FC<Props>`, props ordering, styles co-location and story title casing all conform.
- **The awaited refetch is the interesting design decision** and is done carefully: `throwOnError` is opt-in per query key so that a failing per-row `useJobTaskQuery` (matched by the bare `JOB_TASKS_QUERY_KEY` prefix) cannot close a panel after a submission that succeeded. `Promise.all` attaches handlers to every promise, so the non-first rejections are handled and there is no unhandled-rejection risk.
- **Where it is thin** is the boundary between the new panel state and the *pre-existing* navigation reflexes on the jobs page. The author found and guarded two of them (`useJobQuery`'s `onSuccess`/`onError` in `pages/jobs/index.tsx`) but missed the third — the shared 409 handler — which is Issue 1 below.

---

## Issues Found

### 1. Losing the race for a next job tears the whole Submitted panel down

**[File: apps/creative-portal/components/pages/jobs/hooks.ts]**

> **In plain terms:** You submit a job, the panel offers you three more, and you press Accept on one. Another creative claimed it a moment earlier, so your request is rejected. You get an error message — but instead of leaving you on the panel with the other two jobs still there to pick from, the panel closes completely and you are back on the dashboard with nothing offered. This is precisely the case the feature set out to handle well, and it is the most likely way a creative will first encounter the new panel failing.

**Function/Class:** `handleConflictOrFallback` (lines 87–106), reached from the `updateJob` and `acceptJobMutation` `onError` handlers

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. Log in as a creative with a job open in the side panel (`?jobId=<A>`).
2. Submit job A. The panel switches to the Submitted state and offers up to three next jobs.
3. Have a second creative claim one of the offered jobs (job B) first — or force the `PATCH /jobs` call for job B to return **409**.
4. Press **Accept** on job B's card.
5. **Expected:** the conflict toast appears, job B falls off the list, and the panel stays open with the remaining offered jobs (the PR body's stated design: *"a 409 … leaves the user on the panel rather than on a job that is not theirs"*).
6. **Actual:** the conflict toast appears and the entire side panel closes. The other offered jobs are gone and the user is back on the bare dashboard.

**Problem:** The card's claim goes through the same `useUpdateJobMutation` / `useJobAcceptMutation` error handling as the jobs table, and that handler unconditionally clears the query string on a conflict:

```typescript
const handleConflictOrFallback = (error: unknown, fallback: () => void) => {
  if (isConflictError(error)) {
    showJobConflictError();
    queryClient.invalidateQueries([QUERY_KEYS.JOB_TASKS_QUERY_KEY]);
    queryClient.invalidateQueries([QUERY_KEYS.JOB_SEARCH]);
    queryClient.invalidateQueries([QUERY_KEYS.JOB_CANDIDATES]);

    if (!isTeamProfilePage) {
      replace("", "", { scroll: false });   // <-- closes the Submitted panel
    }

    return;
  }

  fallback();
};
```

The PR guards the two *other* places that clear `?jobId=` while the panel is up — `useJobQuery`'s `onSuccess` and `onError` in `pages/jobs/index.tsx:143` and `:158` both early-return on `submittedPanel.isActive` — but not this one.

**Evidence:** `apps/creative-portal/components/pages/jobs/hooks.ts:97-99` is the offending line. `replace` is `useRouter().replace` (`hooks.ts:77`). `isConflictError` returns true for HTTP 409 and 410 (`components/pages/jobs/utils.ts:247-255`). The teardown chain is fully in-repo and each link is unconditional:

1. `replace("", "", { scroll: false })` clears `?jobId=`.
2. `pages/jobs/index.tsx:38` — `const jobId = router.query.jobId` becomes `undefined`, and is passed straight through as `activeJobId` at `index.tsx:53`.
3. `useSubmittedPanel.ts:36-37` — `isShowingSubmitted = !!submittedJobId && submittedJobId === activeJobId` becomes `false`; the effect at `useSubmittedPanel.ts:81-85` then calls `clearSubmitted()`.
4. `pages/jobs/index.tsx:231` — `isOpen={effectiveJobId !== undefined || submittedPanel.isActive}` is now `false || false`, so `Sidebar` closes.

The existing test `"should leave the user on the panel when the claim is rejected"` (`__tests__/hooks.test.ts:510-529`) drives exactly this 409 path but only asserts `expect(mockPush).not.toHaveBeenCalled()` — it never asserts on `mockReplace`, which is why the defect passes CI.

**Impact:** The feature's headline failure-mode guarantee does not hold. Under contention — which is exactly when three jobs are offered and several creatives are online — the user loses the panel and both remaining offers on a single unlucky click, and has to go back to the dashboard to find work again.

**Fix:** Hold the navigation while the Submitted panel is showing, the same way `pages/jobs/index.tsx` already does for the job query's handlers. `useSubmittedPanel` is declared *after* `handleConflictOrFallback` in the file, so read the flag through a ref (or move the `useSubmittedPanel` call above the mutation declarations):

```typescript
// after the useSubmittedPanel call
const isSubmittedPanelActiveRef = useRef(false);
isSubmittedPanelActiveRef.current = isSubmittedPanelActive;

const handleConflictOrFallback = (
  error: unknown,
  fallback: () => void
) => {
  if (isConflictError(error)) {
    showJobConflictError();
    queryClient.invalidateQueries([QUERY_KEYS.JOB_TASKS_QUERY_KEY]);
    queryClient.invalidateQueries([QUERY_KEYS.JOB_SEARCH]);
    queryClient.invalidateQueries([QUERY_KEYS.JOB_CANDIDATES]);

    // The Submitted panel is not a view of the conflicting job, so
    // there is nothing to bounce off -- and the invalidations above
    // drop the lost job from the offered list on their own.
    if (!isTeamProfilePage && !isSubmittedPanelActiveRef.current) {
      replace("", "", { scroll: false });
    }

    return;
  }

  fallback();
};
```

Then tighten the existing test so the regression is pinned:

```typescript
expect(mockPush).not.toHaveBeenCalled();
expect(mockReplace).not.toHaveBeenCalled();
```

---

### 2. A failed post-submit refresh can close a panel the user has since opened for another job

**[File: apps/creative-portal/components/pages/jobs/hooks.ts]**

> **In plain terms:** You submit a job, and while the page is still quietly catching up you click into a different job. If that catch-up then fails — a slow or flaky connection is enough — the panel you are now reading is closed and you are returned to the dashboard, with no message explaining why.

**Function/Class:** `onActionButtonClick`, `case "submit"` catch block (lines 568–576), calling `useSubmittedPanel`'s `closePanel`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Log in as a creative with job A open in the side panel and submit it.
2. While the post-submit refresh is still in flight — with retries this window is several seconds — click a different job (job B) in the table behind the panel.
3. Have the refresh of the assigned-jobs or job-candidates query fail (offline, 500, or a blocked request).
4. **Expected:** job B's panel stays open; the failed refresh concerns job A's submission, which is no longer on screen.
5. **Actual:** `?jobId=` is cleared and job B's panel closes without a toast or any explanation.

**Problem:** `isPanelJob` is captured once at submission time and stays `true` for the life of the closure, and `closePanel` navigates unconditionally:

```typescript
} catch {
  if (isPanelJob) {
    closePanel();
  }
}
```

```typescript
// useSubmittedPanel.ts:66-69
const closePanel = useCallback(() => {
  setSubmittedJobId(undefined);
  router.replace("", "", { scroll: false });
}, [router]);
```

The author guarded the analogous hazard inside the hook — the auto-close effect at `useSubmittedPanel.ts:91-104` is gated on `isShowingSubmitted`, and the effect at `:81-85` drops a stale id precisely so *"the auto-close below can never fire against someone else's panel"* — but `closePanel` itself carries no such guard, and `hooks.ts` calls it directly.

**Evidence:** `apps/creative-portal/components/pages/jobs/hooks.ts:573-575` calls `closePanel()` guarded only by `isPanelJob`, which is `const isPanelJob = String(jobInfo.id) === activeJobId` evaluated at `hooks.ts:528` — before the `await`. `useSubmittedPanel.ts:66-69` shows `closePanel` always calls `router.replace`. By the time the catch runs, `submittedJobId` has already been cleared by the stale-id effect (`useSubmittedPanel.ts:81-85`) because `activeJobId` moved to job B, so the `setSubmittedJobId(undefined)` is a no-op and only the navigation lands. The sibling `markSettled()` call on the success path is harmless for the same reason — it sets a flag the inactive panel never reads — so the asymmetry is real, not theoretical.

**Impact:** An unexplained panel close during ordinary multitasking on a flaky connection. Low frequency, but it looks like a bug to the user and there is no toast to explain it.

**Fix:** Make `closePanel` a no-op when the panel is not the one on screen — the guard then holds for every caller, not just this one:

```typescript
const closePanel = useCallback(() => {
  // Only ever closes its own panel. The submit path can call this
  // after a slow refetch has failed, by which point the user may
  // have opened a different job.
  if (!isShowingSubmittedRef.current) return;

  setSubmittedJobId(undefined);
  router.replace("", "", { scroll: false });
}, [router]);
```

A ref is needed so `closePanel`'s identity does not churn on every `activeJobId` change (it is a dependency of the auto-close effect). Add a `useSubmittedPanel.test.ts` case: *"does not navigate when the panel has already moved to another job"*.

---

### 3. The submit path — the riskiest new wiring — has no test at the `useJobsPage` level

**[File: apps/creative-portal/components/pages/jobs/__tests__/hooks.test.ts]**

> **In plain terms:** The new "keep the panel open after you submit" behaviour is only tested in two halves that never meet: one test file checks the panel's internal rules in isolation, another checks what happens when you press Accept on a card. Nothing checks the actual moment of submitting — that the panel switches over, that it waits for fresh data, and that it gives up cleanly when the data cannot be loaded. That gap is why Issue 1 above passes the build.

**Function/Class:** `describe("submittedPanel — acting on a next job")` — covers only the card actions

**Severity:** medium

**Confidence:** high

**How to spot it:** This is a code-health and coverage finding, not something a user can reproduce. Open `__tests__/hooks.test.ts` and note that the only `submittedPanel` describe block exercises `onAction`; there is no test that drives `onActionButtonClick(job, "submit", …)`.

**Problem:** Three behaviours introduced by this PR have no coverage where they actually live:

- **`markSubmitted` is called only for the panel's own job.** `const isPanelJob = String(jobInfo.id) === activeJobId` (`hooks.ts:528`) — nothing asserts that submitting some *other* job still calls `replace("", "")` as before.
- **`throwOnError` is applied to the two panel-list keys only.** `hooks.ts:386-389` — nothing asserts that a failing `JOB_TASKS_QUERY_KEY` refetch (a bare prefix matching every table row) leaves the panel open, which is the explicit reason the opt-in exists.
- **`markSettled` vs `closePanel` on the refresh outcome.** `hooks.ts:565-576` — `useSubmittedPanel.test.ts` calls `markSettled` by hand, so nothing verifies `useJobsPage` ever calls it, nor that a rejected refresh closes the panel (validation rule 10).

The existing 409 test (`:510-529`) is also weaker than it reads: it asserts only `mockPush`, which is why it passes despite Issue 1.

**Impact:** The three highest-risk lines of new wiring are unguarded against regression, and one of them is already broken on this branch.

**Fix:** Add a `describe("submittedPanel — submitting")` block alongside the existing one. `mockUpdateJobAsync`, `mockReplace` and `mockQueryClient.invalidateQueries` are already mocked in this file, so the cases are cheap:

```typescript
it("keeps the panel open and marks it settled once the lists refresh", async () => { /* ... */ });
it("closes the panel when a panel-list refetch fails", async () => { /* ... */ });
it("keeps the panel open when only the job-tasks refetch fails", async () => { /* ... */ });
it("closes as before when the submitted job is not the panel's job", async () => { /* ... */ });
```

And tighten the 409 case with `expect(mockReplace).not.toHaveBeenCalled()`.

---

### 4. `parseJobSequence` re-implements the sequence parsing that already exists in `sortJobsByJobSequence`

**[File: apps/creative-portal/utils/findCompletedJob.ts]**

> **In plain terms:** Two separate pieces of code now read the same stored list of job numbers in their own way. Nothing is broken today, but if the stored format ever changes, one will be updated and the other quietly left behind, and jobs will start resolving to the wrong document.

**Function/Class:** `parseJobSequence` (lines 30–35)

**Severity:** medium

**Confidence:** high

**How to spot it:** Code health, not user-reproducible. Compare `findCompletedJob.ts:30-35` with `OrderManagment/partials/OrderJobs/utils.ts:331-333`.

**Problem:** The new helper duplicates the parsing inside the function it is explicitly paired with:

```typescript
// findCompletedJob.ts:30
const parseJobSequence = (jobSequence?: string): number[] =>
  (jobSequence ?? "")
    .split(",")
    .map((jobId) => parseInt(jobId.trim(), 10))
    .filter((jobId) => !Number.isNaN(jobId));
```

```typescript
// OrderJobs/utils.ts:331
const sequence = jobSequence
  .split(",")
  .map((char) => parseInt(char.trim(), 10));
```

`findLatestCompletedJob` calls `parseJobSequence` to decide membership and then hands the *same* string to `orderJobsBySequence` → `sortJobsByJobSequence`, which parses it again. The comment at `findCompletedJob.ts:29` — *"Same comma-separated job-id list `sortJobsByJobSequence` reads"* — is an accurate description of a coupling the code does not enforce. CLAUDE.md's reuse-first convention asks for one implementation.

**Impact:** No current defect — the two parsers agree today (the `NaN` filter only changes membership testing, where a `NaN` could never match a real id). It is a latent divergence on a path that decides which version of a customer's document a creative is shown.

**Fix:** Export the parse from `OrderJobs/utils.ts` and have both call it:

```typescript
// OrderJobs/utils.ts
export const parseJobSequence = (jobSequence?: string): number[] =>
  (jobSequence ?? "")
    .split(",")
    .map((jobId) => parseInt(jobId.trim(), 10))
    .filter((jobId) => !Number.isNaN(jobId));
```

Related nit in the same function: `orderJobsBySequence(...).reverse().find(Boolean)` at `findCompletedJob.ts:143-145` is a roundabout way of taking the last element of a guaranteed-non-empty array — `.at(-1)` says it directly.

---

### 5. The chevron's accessible name identifies the order, not the job it opens

**[File: apps/creative-portal/components/organisms/sidebars/contents/SubmittedNextJobs/partials/NextJobCard/index.tsx]**

> **In plain terms:** Each offered job has a small arrow button that opens it. Someone using a screen reader hears "Open job 198348" — but 198348 is the *order* number, not the job. It reads as a mistake, and if two jobs from the same order are ever offered together, both buttons would announce exactly the same thing with no way to tell them apart.

**Function/Class:** `NextJobCard`, the `IconButton` at line 73

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Submit a job so the Submitted panel appears with at least one offered card.
2. Move through the panel with a screen reader (or inspect the arrow button's `aria-label` in DevTools).
3. **Expected:** a name that identifies what will open — e.g. the work item's subject.
4. **Actual:** `Open job 198348`, where the number is `nextJob.orderId`.

**Problem:**

```typescript
<IconButton
  icon={IconArrowRightTiny}
  aria-label={`Open job ${nextJob.orderId}`}
  onClick={() => onOpenJob(nextJob.id)}
/>
```

The label interpolates `orderId` while the handler navigates with `nextJob.id`. The card does display the order id as its visible identifier, so the mismatch is internally consistent on screen — but the word "job" makes the announced name wrong, and it is not the most useful name available.

**Evidence:** `NextJobCard/index.tsx:73-77`. `nextJob.subject` is already in hand at `:69` (it is passed as the heading's `title` attribute), so a more specific name costs nothing.

**Impact:** Screen-reader users get a misleading control name on the panel's primary navigation affordance. `jsx-a11y` cannot catch this — the label is present, just inaccurate.

**Fix:**

```typescript
aria-label={`Open ${nextJob.subject}`}
```

---

### 6. A failed pay lookup leaves a placeholder that never resolves

**[File: apps/creative-portal/components/organisms/sidebars/contents/SubmittedNextJobs/partials/NextJobCard/hooks.ts]**

> **In plain terms:** Each offered job shows what it pays. If the information that figure is worked out from cannot be loaded, the card shows a grey shimmering placeholder — and keeps showing it indefinitely, with no message and no way to retry. The rest of the card works, so it looks like the page is stuck rather than that something failed.

**Function/Class:** `useNextJobCard`, `isLoadingPay` (lines 62–63)

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Submit a job so the Submitted panel offers at least one card.
2. Make the order lookup for that card's order fail (block `GET /orders/{orderId}`, or use an order the current user cannot read).
3. **Expected:** either a pay figure, or a short indication that it is unavailable.
4. **Actual:** the pay slot shows a shimmering skeleton forever. No toast, no error text, no retry.

**Problem:**

```typescript
isLoadingPay:
  isLoadingJobTasks || isLoadingOrder || !order || !jobTasks,
```

Once the queries settle in an error state, `isLoading` is `false` but `data` is `undefined`, so the expression stays `true` permanently. The reasoning in the comment above it is sound — a figure computed from absent data would be confidently wrong — but "failed" is being rendered as "still loading".

**Evidence:** `NextJobCard/hooks.ts:57-63`, consumed by `NextJobCard/index.tsx:97-108` where `LoadingWrapper isLoading={isLoadingPay}` holds the `SkeletonBox`. The PR's own test `"shows no pay figure when the order it derives from is missing"` (`index.test.tsx:306`) pins this as intended behaviour.

**Impact:** A permanently-loading element on a new, prominent panel. Cosmetic rather than functional — the Accept/Start button still works.

**Fix:** Low priority, and worth noting the table's Pay cell behaves identically (`JobTable/consts.tsx:219-221`), so this is *consistent* with the codebase rather than newly wrong. If it is changed, change both — and prefer distinguishing "errored" from "loading" via the queries' `isError`, rendering a dash or "—" rather than an endless shimmer.

---

### 7. Contradictory duplicated comment on the new `submittedPanel` prop

**[File: apps/creative-portal/components/organisms/sidebars/JobSidebar/types.ts]**

> **In plain terms:** Two different explanations of the same thing sit stacked on top of each other, left over from an edit. Nothing works differently; the next person to read it just has to work out which one is current.

**Function/Class:** `JobSidebarProps`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible. `JobSidebar/types.ts:17-22`.

**Problem:**

```typescript
// PP-2095 req 6.1: after a submission the panel stays open and offers
// the user's next jobs instead of closing.
// PP-2095 req 6.1: present when the page can offer next jobs after a
// submission. Omitted by callers that keep the close-on-submit
// behaviour.
submittedPanel?: SubmittedPanelView;
```

Two comment blocks, both opening `PP-2095 req 6.1:`, describing the prop from two different angles — a copy-paste artefact from a rewrite.

**Impact:** None functional. Noise in a type that two apps' worth of readers will hit.

**Fix:** Keep the second block (it describes the *prop*, not the feature) and drop the first.

---

## Open Questions

- The auto-close effect (`useSubmittedPanel.ts:91-104`) fires whenever `nextJobs` empties *after* the first settle, not only on the settle itself. If the user tabs away and back while all three offered jobs get claimed by others, a window-focus refetch would close the panel under them. Is that the intended reading of req 6.4, or should the close only be evaluated once, immediately after the submission? — `apps/creative-portal/components/pages/jobs/useSubmittedPanel.ts:91`
- `createFileItem` now shows "Reviewed by" / "QA'd by" on the **original** row when `representsCompletedWork` is true, and in the order panel `representsCompletedWork` is `completedWorkItemVersionId === originalWorkItemVersionId` whenever no AI pass resolved. Can the backend ever stamp an order's completed version id equal to its original upload's id on an order where nobody has worked yet? If so, an untouched upload would gain a reviewer credit. (The customer portal is unaffected — it passes neither `reviewerName` nor `qaName`.) — `packages/shared/components/organisms/FilesSection/hooks.ts:126`
- `getAiEditKind`'s legacy fallback picks the service job with `orderJobs.find(job => job.jobType === JobType.SERVICE)` on the unsorted array, whereas the WYSIWYG path it mirrors searches `sortedJobs`. On an order with more than one service job, would the two sides pick the same one? — `apps/creative-portal/api/jobs/utils.ts:61`
- `NextJobCard` calls `getJobActionState` directly in `index.tsx` rather than returning it from its sibling `useNextJobCard`. It is a pure function so it does not breach the "`index.tsx` is UI-only" rule, but the card already has a hook — was keeping it at the call site deliberate? — `apps/creative-portal/components/organisms/sidebars/contents/SubmittedNextJobs/partials/NextJobCard/index.tsx:45`
- `Action/index.tsx` now guards the status badges with `isTerminal && jobTableItem.status === "Submitted"` (and `=== "Canceled"`). Since `isTerminal` is true exactly when the status is one of those two, the conjunct is always redundant — was it added as documentation, or is a future non-`Submitted`/`Canceled` terminal status expected? — `apps/creative-portal/components/molecules/tables/JobTable/Partials/Action/index.tsx:44`

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

The PR body reports 205 tests passing across the affected suites plus 29 in shared `FilesSection`, with `typecheck` and `lint` clean for `@proofed/creative-portal`, and states the **full** suite was not run because the creative-portal vitest run stalls on `develop` independently of this branch. None of that was re-verified here.

Because this PR changes `packages/shared/components/organisms/FilesSection`, a scoped creative-portal run is not sufficient — the **full, unscoped** monorepo suite (or at minimum `@proofed/shared` + `@proofed/customer-portal`) must pass before merge. I confirmed by reading the customer-portal call site (`OrderTable/partials/OrderAndBriefDetails/utils.tsx:141`) that it passes neither `reviewerName` nor `qaName`, so the `representsCompletedWork` change is inert there — but that is a code-reading argument, not a green build.

---

## Tests

- ✅ Tests added for all new units: `selectNextJobs` (9 cases), `useSubmittedPanel` (12), `SubmittedNextJobs` + `NextJobCard` (16), `getAiEditKind`/`hasCompletedPreEdit` (11), `findCompletedJob` (30), `useSourceJobWorkItemVersion` (17), `FilesSection` (8).
- ✅ Assertions are behavioural and requirement-anchored, not snapshot churn — e.g. *"blocks starting an assigned job still behind its predecessor"*, *"withholds the pre-submit list until the refetch settles"*, *"prefers a dated completed job over an undated one"*. This is well above the usual bar for this repo.
- ✅ Edge cases are covered where they were the point of the change: undated completion times, a `jobSequence` accounting for none of the completed jobs, a closed order's `currentJobId: -1`, a failed AI pass, an empty/missing job list.
- ❌ **The submit path has no `useJobsPage`-level test** — see Issue 3. `markSubmitted` gating, the per-key `throwOnError`, and `markSettled` vs `closePanel` are all untested where they are wired.
- ⚠️ **The 409 test is too loose** — `__tests__/hooks.test.ts:510` asserts `mockPush` only, which is why Issue 1 is green on CI.
- ⚠️ **No test pins `closePanel`'s no-op-when-stale behaviour** (Issue 2), because that behaviour does not currently exist.
- ✅ Story present and conformant: `title: "Organisms/Sidebars/Submitted next jobs"` (sentence case, correct hierarchy), CSF3, `withCreativePortalProviders`, cache seeded on the real `ORDER_JOBS_QUERY_KEY`, future-dated deadlines (`2099-…`), realistic data. No external URLs.
- ❌ **Manual testing not done and design fidelity not verified** — the author states this plainly. Figma `31263:8917` has not been compared in a browser.

### Suggested manual QA script

1. **Issue 1 —** Open a job, submit it, and confirm the panel stays open and offers up to three jobs. Get a colleague to claim one of them a moment before you press **Accept** on it (or ask a developer to force a 409). *The panel must stay open with the remaining jobs; only the lost job should disappear.* Today it closes entirely.
2. **Issue 2 —** Submit a job, then immediately click a different job in the table behind the panel while the page is still refreshing. With the network throttled or briefly disconnected, *the second job's panel must stay open.*
3. **Issue 5 —** With a screen reader, tab to the arrow button on an offered card. *It should announce the work item's title, not an order number.*
4. **Issue 6 —** Block the order lookup for an offered card. *Note whether the pay slot shimmers indefinitely* — confirm with the PO whether that is acceptable, given the jobs table already behaves this way.
5. **Req 6.2/6.3 —** Submit with 0, 1, 2, 3 and 4+ jobs available. *Three at most, assigned before queue, each in deadline order; with none available the panel should close.*
6. **Req 6.4 / validation rule 10 —** Submit with the job-candidates request blocked. *The panel should close cleanly and show no "This job was not found." toast.*
7. **Req 6.5 —** Press the arrow on a card without claiming it. *The panel should move to that job.*
8. **Pay parity —** For a job visible in both places, *the figure on the card must match the Pay column in the table exactly*, including the bonus and the "estimated" treatment.
9. **Design fidelity —** Compare the panel side by side with Figma `31263:8917`, including the absence of the coloured accent bar at the top of the sidebar in this state.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ One confirmed high-severity defect in the primary new flow (Issue 1), one medium (Issue 2) |
| Regression risk | ⚠️ Medium — `getJobActionState` and `getDisplayPayAmountForJob` changes are behaviour-preserving for existing callers (verified by reading `Action/index.tsx` and `GroupHeaderPayCell`), but `FilesSection` is a cross-app `packages/shared` export and the full suite has not been run |
| Tests | ⚠️ Strong for the new units, absent for the submit path; the 409 test is too loose to catch Issue 1 |
| Accessibility | ⚠️ One inaccurate `aria-label` (Issue 5); headings, skeleton `aria-hidden` and keyboard affordances are otherwise correct |
| Error handling | ⚠️ Deliberate and well-reasoned on the refresh path; the conflict path is the gap (Issue 1) |
| Security | ✅ No new endpoint, input, `dangerouslySetInnerHTML`, secret or redirect — but `/security` has not been run, which CLAUDE.md requires before a PR |
| Code quality | ✅ Reuse-first, conventions followed, comments explain *why*; two trivial cleanups (Issues 4 and 7) |
| Validation suite | ⏭️ Skipped — user opted out |
| Mergeable state | ✅ Clean (GitHub), but the base is the unmerged `feature/PP-2095-latest-job-copy-file-based-orders` (#2453) |

---

## Recommendation

**Request changes.**

The design is good and the reuse discipline is genuinely above average for this repo — `getJobActionState` and the `withBonus` parameterisation are the right shapes, and the test suite for the new units is strong. But the feature's headline guarantee does not hold in the one failure mode it was built for, and the test that was meant to cover it only checks half the navigation.

Before merge:

1. **Fix Issue 1** — guard `handleConflictOrFallback`'s `replace("", "")` on the Submitted panel being active, and tighten the existing 409 test with `expect(mockReplace).not.toHaveBeenCalled()`.
2. **Fix Issue 2** — make `closePanel` a no-op when the panel has already moved to another job, with a covering test.
3. **Add the submit-path tests (Issue 3)** — `markSubmitted` gating, panel-list-only `throwOnError`, and `markSettled` vs `closePanel`.
4. **Run the full unscoped validation suite** (`test` / `typecheck` / `lint` / `build`). A creative-portal-scoped run is not sufficient because `packages/shared/components/organisms/FilesSection` changed. If the creative-portal vitest run really does stall on `develop`, raise that separately — it should not be absorbed as a standing exception.
5. **Run `/security`** on the branch, as CLAUDE.md requires.
6. **Verify against Figma `31263:8917` in a browser**, per the author's own note — do not approve on design fidelity until that is done.
7. **Get a PO decision on req 6.6** (copy changeable without a deploy) and on the remaining open items the author listed — req 2.1 on the job panel and req 4.3 filename handling.

Then: Issues 4, 5, 6 and 7 are small and can ride along or follow. Merge #2453 first.

Nothing in the PR description, commit messages, code comments or Jira reference attempted to influence this review's process or verdict.
