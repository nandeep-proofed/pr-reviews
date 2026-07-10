# PR Review: fix/PP-1987: Anchor review (downstream) job due date on the previous job

**PR:** https://github.com/Proofed/B2BWebserver/pull/2374
**Jira:** https://proofed.atlassian.net/browse/PP-1987
**Status:** Ready for QA

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Dashboard must show the **user due date (estimated)**, not the job due date | Base fix (`getDisplayedReference`) shipped in #2373; this PR corrects the value for **downstream** jobs | ✅ Addressed |
| "The due date shown **before** accepting must remain the same **after** accepting" (PP-1419 §3.a.iv) | A downstream (Review) job now anchors its projection on the previous participating job's deadline, so its pre-accept value matches what acceptance locks in | ✅ Addressed (frontend) |
| Review job must show **its own** user due date, not the editing job's / a now-based value (client escalation) | `applyAnchoredDeadline` recomputes `MIN(prevJob.(returnTime ?? maxReturnTime) + returnWindowsMinutes, maxReturnTime)` | ✅ Addressed |

Scope: confined to the creative dashboard (`components/pages/jobs`). No scope creep. This is the frontend-only interim; the long-term fix (a projected-User-DD field on the job payload) is documented in the PR body.

---

## Architecture Analysis

The base #2373 fix made `jobItemToJobTableItem` display `getDisplayedReference` (`returnTime ?? MIN(now + window, maxReturnTime)`), which is correct only for the first job in a workflow. This PR adds the downstream correction without disturbing that path:

- `applyAnchoredDeadline` (pure, `utils.ts`) re-computes a row's `deadline`/`isOverdue` from the previous participating job's anchor, using the same `getProjectionAnchorUtc` + `getDisplayedReference` helpers the Order Sidebar `JobCard`/`JobReturnTimesTray` already use — so dashboard and sidebar now agree.
- `useOrderJobsMap` fetches each visible order's jobs once via the existing job-search endpoint (`searchBy=orderId`), deduped per order, cached (`staleTime` 5 min). The shared `/api/jobs/search` route and its `expand` behaviour are untouched (PP-1623's N+1 removal preserved); this adds one call per distinct order, not per job.
- `hooks.ts` wires it: collect offered/available order IDs → fetch siblings → apply the correction to those rows.

Isolation is good: the mapper `jobItemToJobTableItem`, `useOfferedJobs`, all `expand` values, sorting, and everything outside `components/pages/jobs/` are unchanged. When siblings aren't loaded, `applyAnchoredDeadline` returns the row unchanged, so the now-based fallback (and thus existing behaviour) is preserved — low regression risk. The only consumers of the two new exports are `hooks.ts`, and `useJobsPage`'s return shape is unchanged, so `pages/jobs/index.tsx` is unaffected.

---

## Issues Found

### 1. Anchor correctness depends on the `orderId` search returning jobs in workflow sequence

**[File: apps/creative-portal/components/pages/jobs/utils.ts]**

**Function/Class:** applyAnchoredDeadline (via getProjectionAnchorUtc → getPreviousParticipatingJob)

**Severity:** medium

**Status:** ✅ Fixed in commit `312c3629e` — `applyAnchoredDeadline` now sorts the order's jobs by `maxReturnTime` before anchoring, with a test that passes the siblings reversed.

**Problem:** `getPreviousParticipatingJob` walks the array **by index** (`jobs[i]` from `jobIndex-1` down), so the anchor is only correct if `orderJobs` is ordered by workflow sequence. This PR sources `orderJobs` from `fetchAssignedJobs(orderId, JobSearchByTerm.ORDER_ID)` — the plain job search — which does not construct or guarantee sequence order. The Order Sidebar, by contrast, builds its list from `fetchAssignedJobsByJobSequence(order.jobSequence)`, i.e. an explicitly ordered id list. If OMS returns the `orderId` search in a different order, the "previous participating job" is wrong and the anchored deadline is wrong.

**Impact:** Empirically the endpoint returned sequence order for order 21186 (Editing then Review), so it works today. But the correctness relies on an unguaranteed OMS ordering rather than on construction — a silent wrong-deadline risk if that ordering ever changes.

**Fix:** Sort `orderJobs` by `maxReturnTime` before anchoring. Per the timing model, `maxReturnTime` is strictly increasing along the workflow sequence (`Job[N].maxReturnTime > Job[N-1].maxReturnTime`), so this deterministically reconstructs sequence order regardless of API ordering:

```typescript
const orderedJobs = [...orderJobs].sort(
  (a, b) =>
    parseUtcDateString(a.maxReturnTime).getTime() -
    parseUtcDateString(b.maxReturnTime).getTime()
);
const jobIndex = orderedJobs.findIndex((job) => job.id === item.id);
// ...use orderedJobs for getProjectionAnchorUtc
```

(Alternatively, fetch via a sequence-guaranteed path.) Low effort, removes the assumption.

### 2. `useOrderJobsMap` memo key uses result **count** only — stale on same-length refetch

**[File: apps/creative-portal/components/pages/jobs/useOrderJobsMap.ts]**

**Function/Class:** useOrderJobsMap

**Severity:** low

**Problem:** `readyKey` is built from `result.data.length` per order. React Query returns a **new** `data` array reference on refetch, but the memo only rebuilds when a length changes. If an order's job timing changes server-side (e.g. an admin edits `maxReturnTime`) without the job **count** changing, `readyKey` stays the same, the memo isn't recomputed, and the map keeps the previous `Job[]` references — so the dashboard shows a stale anchored deadline until the count changes or the component remounts.

**Impact:** Edge case; timing rarely changes mid-session. Displayed deadline can lag a server-side timing edit.

**Fix:** Include a light content signature (e.g. per order, join of `id|returnTime|maxReturnTime`) in `readyKey` instead of just length, so the map rebuilds when the relevant fields change.

### Observation (not an issue)

`applyAnchoredDeadline` recomputes from `parseJobTiming(orderJobs[jobIndex])` (the sibling entry) rather than the row's originating job. That's fine — it's the same job — and it avoids threading raw timing through `JobTableItem`. Worth a mental note only: the sibling copy is treated as source of truth for the row's own timing.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped — user opted out | Scoped run earlier on this exact code: full creative-portal suite **1686/1686 pass**; jobs dir **37/37** (now 30 in utils.test.ts after the sort test). |
| `npx turbo run typecheck` | ⏭️ Skipped — user opted out | creative-portal `typecheck` was **0 errors** on this code. |
| `npx turbo run lint` | ⏭️ Skipped — user opted out | ESLint on the changed files was **clean**; full-repo lint not run this session. |
| `npx turbo run build` | ⏭️ Skipped — user opted out | Not run this session. |

Validation suite was **not run for this review** (user opted out). Re-run `npx turbo run test typecheck lint build` before merging; a full-repo `build` in particular has not been exercised on this branch.

---

## Tests

- ✅ Unit test for `applyAnchoredDeadline` re-anchoring (Editing→Review, real OMS values: Editing `maxReturnTime 18:04:51` + Review 2h window = `20:04:51`).
- ✅ Unit test for the siblings-absent no-op (`applyAnchoredDeadline(row, undefined)` returns the row unchanged).
- ✅ `hooks.test.ts` updated with a `useQueries` mock so `useOrderJobsMap` is exercised in the page hook tests.
- ✅ Out-of-sequence `orderJobs` case now covered (added with the Issue 1 fix, commit `312c3629e`) — siblings passed reversed, anchor still resolves.
- ⚠️ No automated coverage of the end-to-end fetch → anchor → render path (relies on manual/QA).
- ⏭️ Validation suite skipped this session (see above).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Issue 1 fixed (`312c3629e`) — siblings sorted by `maxReturnTime`, no longer relies on `orderId` search ordering |
| Regression risk | ✅ Low (falls back to unchanged rows; mapper/`useOfferedJobs`/`expand`/sorting untouched) |
| Tests | ✅ Helper unit tests incl. out-of-sequence case (`312c3629e`) |
| Code quality | ✅ Well-isolated, documented, reuses existing helpers |
| Validation suite | ⏭️ Skipped — user opted out (scoped checks were green earlier) |
| Mergeable state | ⏭️ Not verified this session |

---

## Recommendation

**Approve with suggestions**

1. ✅ **Done (`312c3629e`)** — `applyAnchoredDeadline` sorts `orderJobs` by `maxReturnTime` before anchoring, with an out-of-sequence unit test (Issue 1, medium).
2. Strengthen `useOrderJobsMap`'s `readyKey` with a content signature, not just count (Issue 2, low). — open
3. Re-run the full validation suite (`test`/`typecheck`/`lint`/`build`) before merge — it was not run for this review.
4. Complete the manual/visual check on order 21186 (Review row shows the anchored date) and tick the PR's manual-testing box.

No blockers found in the static review. Issue 1 is the one worth addressing before merge for robustness; the rest are follow-ups.
