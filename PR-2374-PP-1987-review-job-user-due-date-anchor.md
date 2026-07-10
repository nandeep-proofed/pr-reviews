# PR Review: fix/PP-1987: Anchor review (downstream) job due date on the previous job

**PR:** https://github.com/Proofed/B2BWebserver/pull/2374
**Jira:** https://proofed.atlassian.net/browse/PP-1987
**Status:** Ready for QA
**Reviewed at:** HEAD `198284bf0` (regenerated after review-round changes)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Dashboard shows the **user due date (estimated)**, not the job due date | Base fix (`getDisplayedReference`) shipped in #2373; this PR corrects the value for **downstream (Review)** jobs | ✅ Addressed |
| "Due date shown **before** accepting stays the same **after** accepting" (PP-1419 §3.a.iv) | A Review row now anchors its projection on the previous participating job, so its pre-accept value matches what acceptance locks in | ✅ Addressed (frontend) |
| Review job must show **its own** user due date, not the editing job's / a now-based value (client escalation) | `applyAnchoredDueDate` recomputes `MIN(prevJob.(returnTime ?? maxReturnTime) + returnWindowsMinutes, maxReturnTime)` | ✅ Addressed |

Frontend-only interim; the long-term fix (a projected-User-DD field on the job payload) is documented in the PR body.

---

## Architecture Analysis

Final shape after the review rounds:

- **`utils.ts` → `applyAnchoredDueDate`** (pure helper): given a row and its order's jobs, re-computes `deadline`/`isOverdue` by anchoring on the previous participating job (`getProjectionAnchorUtc` + `getDisplayedReference` — the same helpers the Order Sidebar uses). Returns the row unchanged when siblings are absent, preserving the now-based fallback from `jobItemToJobTableItem` (which is itself untouched).
- **`services/jobs/search` → `useJobSearchQueries`**: generalized to accept a `searchBy` (defaults to `proofedUserId`; no other callers). It's the existing single-`useQueries` multi-fetch hook — same structure as `useAvailableJobs`.
- **`hooks.ts`**: collects **Review** order IDs, calls `useJobSearchQueries(orderIds, undefined, ORDER_ID)` once, derives an `orderId → jobs` map in a `useMemo`, and re-anchors only the Review rows.

This resolves the earlier review points: no custom `useOrderJobsMap` hook (removed), no `readyKey`/`eslint-disable`, no client-side sort, no `staleTime`, "due date" naming, and the shared `/api/jobs/search` route + `expand` values are untouched (PP-1623 preserved). Consumption mirrors the established `useAvailableJobs` → `availableJobTableItems` pattern. Regression risk is low: `useJobSearchQueries`'s signature change has no other consumers, `jobItemToJobTableItem`/`useOfferedJobs` are unchanged, and `useJobsPage`'s return shape is unchanged (so `pages/jobs/index.tsx` is unaffected).

---

## Issues Found

### 1. Anchor relies on the `orderId` search returning jobs in workflow sequence

**[File: apps/creative-portal/components/pages/jobs/utils.ts]**

**Function/Class:** applyAnchoredDueDate (via getProjectionAnchorUtc → getPreviousParticipatingJob)

**Severity:** low

**Status:** ✅ ACCEPTED — won't fix. The team has accepted this as a known assumption (relies on the OMS `orderId`-search returning `jobSequence` order). No code change.

**Problem:** `getPreviousParticipatingJob` walks the array by index, so the anchor is only correct if `orderJobs` is in workflow sequence. The per-order search (`searchBy=orderId`) is relied on to return that order — and `getAssignedJobs` applies no client/BFF-side sort, so the order is purely whatever OMS returns. A `maxReturnTime` sort was added and then intentionally removed on the basis that the search returns jobs in sequence.

**Impact:** Correct for the current 2-job Editing→Review orders (created in order, so id-order == sequence-order). Latent risk only if OMS ever returns the `orderId` search in a non-sequence order — most plausibly for post-live-inserted / 3+ step orders (PP-1863), where a downstream job could anchor on the wrong previous job.

**Decision:** Accepted for this hotfix. The assumption is documented in the `applyAnchoredDueDate` code comment. If it ever needs hardening, sort `orderJobs` by `maxReturnTime` (strictly increasing along the sequence, preserved through insertion) before anchoring — deterministic and independent of OMS order. Optional de-risk: confirm the OMS `orderId`-search ordering contract with the backend.

### 2. "Downstream" is approximated by `jobType === REVIEW`

**[File: apps/creative-portal/components/pages/jobs/hooks.ts]**

**Function/Class:** useJobsPage (orderIds + anchored* memos)

**Severity:** low

**Problem:** The re-anchor (and the sibling fetch) are scoped to `jobType === JobType.REVIEW`. On this dashboard that's exactly the set of downstream jobs (Editing is always first; RETURN/QA are filtered out), and scoping avoids a wasted fetch for Editing-only orders. But job type is a proxy for "not the first participating job" — if a workflow ever shows a non-Review downstream job on this dashboard, it wouldn't be re-anchored.

**Impact:** None for current workflows. A future workflow change with a different downstream job type shown here would silently miss the correction.

**Fix:** Acceptable as-is (documented in the code comment). Revisit if the dashboard ever surfaces a non-Review downstream job.

### Observation (not an issue)

`orderJobsMap` (and the downstream `anchored*` memos) recompute each render because `useJobSearchQueries` returns a fresh array each render. This is consistent with the existing `availableJobTableItems` memo (which depends on `availableJobsQueriesResults` the same way); the per-render cost is a small map build over a handful of orders. The `readyKey` micro-optimization from an earlier revision was removed on review to match the codebase pattern.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped — user opted out | Jobs dir green on this HEAD: `utils.test.ts` + `hooks.test.ts` = **37/37**. Full suite was **1686/1686** on the equivalent logic earlier. |
| `npx turbo run typecheck` | ⏭️ Skipped — user opted out | creative-portal `typecheck` **0 errors** on this HEAD. |
| `npx turbo run lint` | ⏭️ Skipped — user opted out | ESLint clean on changed files on this HEAD; full-repo lint not run. |
| `npx turbo run build` | ⏭️ Skipped — user opted out | Not run this session. |

Validation suite was **not run for this review** (user opted out). Re-run `npx turbo run test typecheck lint build` before merging; a full-repo `build` has not been exercised on this branch.

---

## Tests

- ✅ `applyAnchoredDueDate` re-anchor test (Editing→Review, real OMS values: `18:04:51 + 2h = 20:04:51`).
- ✅ `applyAnchoredDueDate` siblings-absent no-op test.
- ✅ `hooks.test.ts` mock updated (`useJobSearchQueries: () => []`).
- ⚠️ No automated coverage of the end-to-end fetch → anchor → render path (relies on manual/QA).
- ⏭️ Validation suite skipped this session.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes the downstream (Review) due date; sound for current data (Issue 1 accepted) |
| Regression risk | ✅ Low (mapper/`useOfferedJobs`/`expand`/sorting untouched; generalized hook has no other callers) |
| Tests | ✅ Helper unit tests; ⚠️ no e2e |
| Code quality | ✅ Idiomatic (`useJobSearchQueries` + derive-in-component); prior review points addressed |
| Validation suite | ⏭️ Skipped — user opted out (scoped checks green on this HEAD) |
| Mergeable state | ⏭️ Not verified this session |

---

## Recommendation

**Approve with suggestions**

1. Re-run the full validation suite (`test`/`typecheck`/`lint`/`build`) before merge — not run for this review.
2. Complete the manual/visual check on order 21186 (Review row shows the anchored date) and tick the PR's manual-testing box.
3. Issue 1 (workflow-sequence assumption) is **accepted — won't fix** for this hotfix; keep it (and the `jobType === REVIEW` proxy, Issue 2) in mind if OMS ordering or the dashboard's visible job types change.

No blockers in the static review. Both items are low-severity, team-accepted trade-offs already captured in code comments; Issue 1 is explicitly accepted / won't fix.
