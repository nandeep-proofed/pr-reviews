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

**Problem:** The re-anchor (and the sibling fetch) are scoped to `jobType === JobType.REVIEW`, a proxy for "not the first participating job". The dashboard filters out RETURN/QA but **not** SERVICE, REVIEW, or **AI** (AI routes to the EDITING queue). So if a non-Review downstream job — most plausibly an **AI** "post-edit" step — is ever offered/available to a creative here, it would not be re-anchored and would show the now-based value (the same PP-1987 bug, uncaught).

**Impact:** None for the current Editing→Review workflow. Would bite if a non-Review downstream job (e.g. AI) surfaces offered/available on this dashboard.

**Fix:** Open — confirm with the team whether an AI (or any non-Review) job can be offered/available to a creative on this dashboard. If no, accept as documented. If yes, gate on position ("not the first participating job") instead of job type, or drop the `REVIEW` gate (accepting the extra Editing-only-order fetch).

### Observation (not an issue)

`orderJobsMap` (and the downstream `anchored*` memos) recompute each render because `useJobSearchQueries` returns a fresh array each render. This is consistent with the existing `availableJobTableItems` memo (which depends on `availableJobsQueriesResults` the same way); the per-render cost is a small map build over a handful of orders. The `readyKey` micro-optimization from an earlier revision was removed on review to match the codebase pattern.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `test` (ticket + creative-portal) | ✅ Pass | PP-1987 ticket tests **92/92** (`utils`, `hooks`, `jobItemToJobTableItem`, `dynamicReturnTimes`, `postAcceptJob`); `@proofed/creative-portal` green. |
| `npx turbo run test` (whole repo) | ⚠️ 1 pre-existing unrelated failure | `@proofed/shared` → `utils/formatWordQuantity.test.ts` (`10,00,000` vs `1,000,000`) — an `Intl` **locale artifact** of the test machine, not touched by this PR and pre-existing on `develop`. |
| `npx turbo run typecheck` | ✅ Pass | **0 errors** across all 5 workspaces. |
| `npx eslint` (changed files) | ✅ Pass | Clean on all files this PR changes. (Full-repo lint has an unrelated pre-existing prettier error in `JobReturnTimesTray/index.test.tsx`.) |
| `npx turbo run build` | ⏭️ Not run | Not exercised this session. |

Everything **this PR changes** is clear: ticket tests 92/92, creative-portal green, typecheck 0 errors, changed-file lint clean. The only whole-repo red is the pre-existing `@proofed/shared` locale test (unrelated). Re-run a full `build` before merging.

---

## Tests

- ✅ `applyAnchoredDueDate` re-anchor test (Editing→Review, real OMS values: `18:04:51 + 2h = 20:04:51`).
- ✅ `applyAnchoredDueDate` siblings-absent no-op test.
- ✅ `hooks.test.ts` mock updated (`useJobSearchQueries: () => []`).
- ✅ Ran: PP-1987 ticket tests **92/92**; typecheck **0 errors** (all workspaces); changed-file lint **clean**.
- ⚠️ Whole-repo `test` has one pre-existing, unrelated failure in `@proofed/shared` (locale artifact); full `build` not run this session.
- ⚠️ No automated coverage of the end-to-end fetch → anchor → render path (relies on manual/QA).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes the downstream (Review) due date; sound for current data |
| Regression risk | ✅ Low (mapper/`useOfferedJobs`/`expand`/sorting untouched; generalized hook has no other callers) |
| Tests | ✅ Helper unit tests; ⚠️ no e2e |
| Code quality | ✅ Idiomatic (`useJobSearchQueries` + derive-in-component); prior review points addressed |
| Validation suite | ✅ test (ticket 92/92) + typecheck (0 errors) + changed-file lint clean; ⚠️ 1 pre-existing unrelated `@proofed/shared` locale test fails; build not run |
| Mergeable state | ⏭️ Not re-verified this session |

---

## Recommendation

**Approve with suggestions**

1. `test` (ticket 92/92), `typecheck` (0 errors), and changed-file `lint` are green; run a full `build` before merge (not exercised this session). Note the pre-existing `@proofed/shared` locale test failure is unrelated.
2. Complete the manual/visual check on order 21186 (Review row shows the anchored date) and tick the PR's manual-testing box.
3. Issue 1 (workflow-sequence assumption) is **accepted — won't fix** for this hotfix. Issue 2 (`jobType === REVIEW` scoping) is **open** — confirm whether a non-Review downstream job (e.g. AI) can appear offered/available here before accepting it.

No blockers in the static review. Issue 1 is explicitly accepted / won't fix; Issue 2 needs a quick product confirmation but has no impact on the current Editing→Review workflow.
