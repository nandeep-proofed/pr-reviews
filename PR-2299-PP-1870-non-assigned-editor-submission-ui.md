# PR Review: fix/PP-1870: Hide submission UI for editors viewing a job already accepted by another editor

**PR:** https://github.com/Proofed/B2BWebserver/pull/2299
**Jira:** https://proofed.atlassian.net/browse/PP-1870
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Non-assigned editor opening an already-accepted job link must NOT see the in-progress submission UI (download / upload edited copy / upload track-changes) | Side panel never mounts when `isJobTakenByAnotherEditor` trips (`effectiveJobId` becomes `undefined`), so the Submission UI cannot render | ✅ Addressed |
| Show the offer overview (length, deadline, Accept) **or** an appropriate "this job has been taken / not-available" state | Implements the "not-available" branch: clears the `?jobId=` param and shows a 10s warning toast ("This job is no longer available… accepted by another editor"). User lands back on the job list. Does not render an in-page persistent "taken" card, but that is a reasonable reading of the AC since the job is no longer Offered to them. | ✅ Addressed |
| Assigned editor must still see the submission UI | `proofedUserId === userId` short-circuits the gate → panel opens normally | ✅ Addressed |
| Offer still genuinely available (Offered status) must remain acceptable | `status === "Offered"` short-circuits the gate | ✅ Addressed |

**Scope creep / beyond-Jira:** The PR description claims the gate is scoped to **Editor-only** viewers via a helper `isEditorOnlyRole`, leaving Reviewers and Editor+Reviewer dual-role users untouched. **The shipped code does the opposite** — it gates *every* viewer of `/jobs` (Editor, Reviewer, and dual-role alike). This is broader than the Jira ticket, which is specifically about editors. See Issue #1.

---

## Architecture Analysis

The fix moves the decision up from the side panel (`JobManagement`, which rendered purely off `job.status`) to the page (`JobsPage`). On `/jobs`, when a `?jobId=` is present, it pre-fetches the job with `useJobQuery(jobId)` — the same React Query key (`[JOBS_QUERY_KEY, jobId]`, no `expand`) the panel's own `useJobSidePanel` uses, so this is a genuine cache warm rather than a duplicate request. A pure helper `isJobTakenByAnotherEditor({ job, userId })` decides whether the viewer is locked out (assigned to someone else + past Offered). When it trips, a `useEffect` clears the URL and toasts, guarded by a `useRef` to avoid StrictMode double-toasts. The panel is gated through a new `effectiveJobId` that is `jobId` only once the job has loaded and the gate has not tripped.

The approach is sound and fixes the **root cause** (the panel never knew who the viewer was). The pure-helper-plus-page-effect split is clean and testable. However, the implementation and the PR description have diverged significantly, and routing the panel through "open only after the prefetch resolves" introduces two behavioural regressions for the non-bug path (see Issues #2 and #3).

One positive: because "Assigned to you" rows are fetched by `proofedUserId === userId` and "Available" rows are always `Offered`, the gate can never trip from normal in-app row clicks — only from a stale deep link. So there are no false positives in ordinary navigation.

---

## Issues Found

### 1. PR description describes a role-scoped design that does not exist in the code

**[File: apps/creative-portal/components/pages/jobs/index.tsx]**
**Function/Class:** JobsPage (page-level gate)
**Severity:** high
**Problem:** The PR description's "Fix", "Role scoping", "Behaviour by viewer role" table, and "Testing" sections all describe an `isEditorOnlyRole(user.roles)` helper that gates the prefetch (`enabled: isEditorOnly`), an `isJobTakenByAnotherEditor({ job, userId, isEditorOnly })` signature, and a dual-role bypass (`useJobQuery` `enabled: false` for Editor+Reviewer). **None of this is in the shipped code:**
- `isEditorOnlyRole` does not exist anywhere in the repo (grep returns nothing; the diff to `JobManagement/utils.ts` only adds `isJobTakenByAnotherEditor`).
- `useJobQuery(jobId, { retry: false, enabled: !!jobId })` runs for **every** viewer, not just Editor-only ones.
- `isJobTakenByAnotherEditor` takes `{ job, userId }` only — no `isEditorOnly` parameter.
- The inline code comment even states the opposite of the description: *"Applies to every viewer on /jobs — Editor only, Reviewer only, and Editor + Reviewer alike."*
- The description claims "14 new tests… incl. an explicit Editor + Reviewer → false case so the dual-role bypass cannot regress." The actual test file has **8** tests, **none** for `isEditorOnlyRole`, and **no** dual-role case — because there is no role parameter to test.

**Impact:** Reviewers are approving against a description that misrepresents the runtime behaviour. The promised "dual-role users behave exactly as before" guarantee is unbacked — there is no role scoping and no test protecting it. Anyone later refactoring will trust a "bypass" that isn't there. This is a reviewability/correctness-of-record problem even though the shipped gate happens to be safe for normal navigation.
**Fix:** Reconcile the two. Either (a) update the PR description (and the variable already renamed to `isJobTakenForCurrentViewer`/comment) to state plainly that the gate applies to all `/jobs` viewers and drop the `isEditorOnlyRole`/14-tests/dual-role-bypass claims; or (b) actually implement the role scoping the description promises. Given the analysis below, (a) + the targeted fixes in #2–#4 is the lighter path — but the description must stop claiming a design that isn't shipped.

### 2. "Job not found" handling is lost for invalid/deleted job deep links

**[File: apps/creative-portal/components/pages/jobs/index.tsx]**
**Function/Class:** JobsPage — `effectiveJobId` gating vs `useJobSidePanel` `onError`
**Severity:** medium
**Problem:** Pre-PR, `activeJobId={jobId}` mounted the panel immediately, so `useJobSidePanel`'s `useJobQuery(jobId, { retry: false, onError: … })` would, on a 404/network error, clear the URL and show *"This job was not found. Check the link and try again."* Post-PR, `effectiveJobId` requires `!!targetJob`. When the page-level prefetch errors, `targetJob` stays `undefined`, the panel never mounts, and `useJobSidePanel` never runs — so its `onError` becomes effectively dead code. The page-level query has `retry: false` but **no `onError`**, so the failure is swallowed silently.
**Impact:** Opening `/jobs?jobId=<deleted-or-invalid-id>` (a real case — these are old Slack links) now shows nothing, with the stale `?jobId=` left in the URL, instead of the previous error toast + URL clear. Regression in deep-link error UX.
**Fix:** Add an `onError` to the page-level query that mirrors the old not-found handling, e.g.:

```typescript
const { data: targetJob } = useJobQuery(jobId, {
  retry: false,
  enabled: !!jobId,
  onError: () => {
    router.replace("", "", { scroll: false });
    showToast({
      type: "error",
      title: "This job was not found.",
      text: "Check the link and try again."
    });
  }
});
```

### 3. Side panel no longer opens immediately — every job open now waits on the prefetch

**[File: apps/creative-portal/components/pages/jobs/index.tsx]**
**Function/Class:** JobsPage — `effectiveJobId` / `<JobSidebar isOpen=… />`
**Severity:** medium
**Problem:** `effectiveJobId = jobId && !!targetJob && !isJobTakenForCurrentViewer ? jobId : undefined`. Because it requires `!!targetJob`, `isOpen` is now `false` until the single-job fetch resolves. The clicked row's data comes from the *list* endpoints (`useJobSearch` / `useAvailableJobs`), not from `[JOBS_QUERY_KEY, jobId]`, so the per-job query is a fresh network round-trip on first open. Previously the panel slid open instantly and showed its own "Loading…" state.
**Impact:** A perceptible "nothing happens, then the panel opens" delay on **every** row click for **every** viewer — not just the bug path. The PR description specifically promises dual-role/reviewer flows open "without any pre-validation delay"; in the shipped code they all incur it. This is the cost of gating `isOpen` on the prefetch for all viewers.
**Fix:** Decouple "panel is open" from "gate has cleared." Keep `isOpen` tied to `jobId` and only suppress the *content*/unmount when the gate actually trips, e.g. open the sidebar as before and let it show its loader, while the gate effect closes it + toasts if `isJobTakenForCurrentViewer`. At minimum, open on `jobId` and gate `activeJobId` separately so the loader UX returns. Confirm the chosen behaviour against the assigned-editor happy path so it still feels instant.

### 4. Toast always says "another editor", even when a reviewer took a review job

**[File: apps/creative-portal/components/pages/jobs/index.tsx]**
**Function/Class:** JobsPage — gate `useEffect`
**Severity:** low
**Problem:** Since the gate now applies to all viewers, a Reviewer (or dual-role user) opening a stale link to a `Review` job accepted by **another reviewer** will trip the gate and see `showJobConflictError(UserRole.Editor)` → *"It has already been accepted by another editor."* The newly-added role parameter on `showJobConflictError` is hardcoded to `Editor` and never reflects the job's actual type/taker.
**Impact:** Wrong noun for the reviewer case. Minor user confusion; ironic given the whole point of adding the `acceptedByRole` parameter was to differentiate the two messages.
**Fix:** Derive the role from the job, e.g. `showJobConflictError(targetJob?.jobType === JobType.REVIEW ? UserRole.Reviewer : UserRole.Editor)`. (Or, if the team really wants editor-only behaviour, scope the gate as the description claims and this becomes moot.)

### 5. Helper name `isJobTakenByAnotherEditor` is misleading now that it gates reviewers too

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/utils.ts]**
**Function/Class:** isJobTakenByAnotherEditor
**Severity:** low
**Problem:** The helper is applied to every viewer (including reviewers on review jobs), and the consuming memo is already named `isJobTakenForCurrentViewer`, but the helper is still named `…ByAnotherEditor`.
**Impact:** Naming drift; a future reader assumes it is editor-specific and may reuse it incorrectly.
**Fix:** Rename to something role-neutral, e.g. `isJobTakenByAnotherUser`, and update the 8 tests' `describe` block. Cheap, improves clarity.

---

## Tests

- ✅ Pure helper `isJobTakenByAnotherEditor` is well covered — 8 cases (assignment mismatch, viewer-is-assignee, Offered carve-out, `proofedUserId === 0` sentinel, missing `userId`, and Submitted/Assigned/Approved terminal cases).
- ✅ `showJobConflictError` role-parameter behaviour covered — default (reviewer), explicit reviewer, explicit editor messages.
- ❌ **No test for the actual new behaviour** — the page-level gate (`effectiveJobId`, the toast `useEffect`, the `notifiedUnavailableJobIdRef` dedupe, the URL clear). The most consequential change in the PR is untested. `pages/jobs/__tests__/hooks.test.ts` was not extended.
- ❌ No regression test for the lost not-found path (Issue #2) or the open-delay (Issue #3).
- ⚠️ Description claims 14 tests incl. an Editor+Reviewer dual-role case; reality is 8 helper tests + 3 toast tests and **no** role/dual-role test. Per CLAUDE.md ("Every PR must include tests for new code"), the page-level logic needs coverage — a component/hook test asserting the panel stays closed + toast fires for a not-the-assignee non-Offered job, and stays open for the assignee.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Gate logic is correct for the bug path, but description ≠ implementation and two non-bug regressions exist |
| Regression risk | ⚠️ Medium — lost not-found handling (#2) + delayed panel open for all viewers (#3) |
| Tests | ⚠️ Helper well covered; the actual page-level gate is untested; description over-claims test count |
| Code quality | ⚠️ Clean helper, but misleading name (#5) and hardcoded toast role (#4) |
| Mergeable state | ✅ Clean (per GitHub `mergeable_state`) |

---

## Recommendation

**Request changes**

1. **Reconcile the PR description with the shipped code (#1).** Either delete the `isEditorOnlyRole` / dual-role-bypass / "14 tests" claims and state that the gate applies to all `/jobs` viewers, or actually implement the role scoping described. Do not merge with a description that contradicts the code.
2. **Restore the not-found handling (#2)** by adding `onError` (URL clear + toast) to the page-level `useJobQuery`.
3. **Stop delaying every panel open (#3)** — keep `isOpen` tied to `jobId` and gate only the content/unmount, so the assigned-editor happy path opens instantly with its loader.
4. **Add a test for the page-level gate** (panel stays closed + toast for a non-assignee non-Offered job; opens for the assignee). Required by the project's "tests for new code" rule.
5. **Minor:** derive the toast role from the job type (#4) and rename `isJobTakenByAnotherEditor` → role-neutral (#5).
