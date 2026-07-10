# PR Review: fix/PP-1987: Show user due date (estimated) on creative job dashboard

**PR:** https://github.com/Proofed/B2BWebserver/pull/2373
**Jira:** https://proofed.atlassian.net/browse/PP-1987
**Status:** Ready for Sprint (Bug, Highest priority)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Dashboard must display the **user due date (estimated)**, not the job due date | `jobItemToJobTableItem` now uses `getDisplayedReference` (`returnTime ?? MIN(now + returnWindowsMinutes, maxReturnTime)`) instead of `getEffectiveDeadline` (`returnTime ?? maxReturnTime`) | ✅ Addressed |
| The date shown **before** accepting must remain the same **after** accepting | Display now resolves to the projected estimate for unassigned jobs and the locked `returnTime` once assigned — both express the same estimate, so the value no longer jumps to the hard stop | ✅ Addressed (frontend); depends on OMS persisting the computed `returnTime` — see Issue 3 |
| Behaviour must follow PP-1419 §3.a.iv ("before acceptance: display Window; after acceptance: DynamicReturnTime") for the Job-User view | Matches the helper the admin `JobCard` already uses; unassigned → projected window, assigned → `returnTime` | ✅ Addressed |
| Assigned jobs unchanged (overdue rows stay correct) | Both helpers return `returnTime` when set, so assigned display/overdue are byte-for-byte identical | ✅ Addressed |

No scope creep — the change is confined to the one field-selection helper and its tests.

---

## Architecture Analysis

The dashboard (`pages/jobs` → `JobTable`) maps each `Job` to a `JobTableItem` via `jobItemToJobTableItem`, whose `deadline` field feeds both the Deadline column (`DeadlineDisplay`) and the internal row ordering (`sortAvailableJobs` / `sortAssignedJobs`). The bug was a single wrong helper: `getEffectiveDeadline` falls back to `maxReturnTime` (the hard stop / "job due date") for unassigned jobs, whereas the spec (and the sibling admin `JobCard`) require the projected user due date. The fix swaps to `getDisplayedReference`, which the rest of the dynamic-return-time surfaces (`JobReturnTimesTray`, admin `JobCard`) already consume — so the change increases consistency rather than introducing a new pattern.

Both helpers are pure and live in `api/utils/jobs/dynamicReturnTimes.ts`. The only behavioural delta is in the **unassigned** branch (projected window vs hard stop); the assigned branch is unchanged. Regression surface is limited to this one mapper — no other production code path was touched (verified: `jobItemToJobTableItem` is the sole caller affected, and `getDisplayedReference` already had four other consumers).

The `deadline` field doubling as the sort key means the internal ordering of unassigned rows now keys off the projected window instead of the hard stop. This table has **no user-facing sort control** (confirmed via UI screenshot — plain column headers), and no ticket governs this table's ordering (PP-1644 governs the admin order table), so the reorder is harmless.

---

## Issues Found

### 1. Orphaned comment fragment left above `isOverdue`

**[File: apps/creative-portal/components/pages/jobs/utils.ts]**

**Function/Class:** jobItemToJobTableItem

**Severity:** low

**Status:** ✅ Fixed in commit `de4f62591`.

**Problem:** Line 68 contained a dangling comment fragment `// deadline can't be overdue.` — the tail of a multi-line comment whose first three lines were removed (a format/save step trimmed the intended replacement). On its own it was grammatically incomplete and no longer explained the `isOverdue` logic.

**Impact:** Cosmetic only — no runtime effect. Slightly misleading for future readers.

**Fix:** Restored the full comment:

```typescript
    payBonus: jobItem.payBonus,
    // The displayed deadline above already encodes the assigned-vs-
    // unassigned rule (locked returnTime, else the projected window), so
    // overdue is simply that deadline being in the past. A job with no
    // deadline can't be overdue.
    isOverdue: hasDeadline && isPast(displayedDeadline),
```

### 2. `getEffectiveDeadline` is now unused in production

**[File: apps/creative-portal/api/utils/jobs/dynamicReturnTimes.ts]**

**Function/Class:** getEffectiveDeadline

**Severity:** low

**Status:** ✅ Fixed in commit `de4f62591` (annotated, not removed).

**Problem:** After this PR, `jobItemToJobTableItem` was the last production consumer of `getEffectiveDeadline`. A whole-repo grep now shows only its definition (line 39) plus a unit test (`OrderJobs/utils.test.ts`) — no production call site remains.

**Impact:** Dead production code. Not a defect, but it would linger and mislead ("this must be the effective-deadline/sort helper") unless removed or explicitly reserved. PP-1644's `effectiveDeadline = returnTime ?? maxReturnTime` sort key for the admin order table is the natural future consumer, so keeping it is intentional.

**Fix:** Kept the helper (it is exported and unit-tested) and added a comment noting that user-facing display uses `getDisplayedReference`, while this coalesced value is reserved for the PP-1644 admin order-table sort key — so it no longer reads as dead code.

### 3. Fix guarantees correct display only if OMS persists the computed `returnTime`

**[File: apps/creative-portal/components/pages/jobs/utils.ts]**

**Function/Class:** jobItemToJobTableItem (interaction with the accept flow)

**Severity:** low (documented; likely out of this repo's scope)

**Problem:** The display now shows the projected estimate even when `returnTime` is null, so it is robust. But the *"before == after accept"* guarantee ultimately relies on the accept flow (`postAcceptJob` → `computeReturnTime`) persisting a `returnTime` that equals the pre-accept projection. If OMS does not store the value sent on candidate accept, the after-accept row falls back to the (still-correct) projected window, which will drift slightly as `now` advances.

**Impact:** Edge case; the user-visible symptom (showing the hard stop) is resolved regardless. Any residual mismatch would be a backend persistence issue, not this display path.

**Fix:** None in this PR. Flagged in the PR description already; worth a QA note to confirm OMS persists the accept-time `returnTime`.

### Observation (not an issue): unassigned job past its hard stop still flags overdue

An unassigned job whose `maxReturnTime` is in the past projects to `maxReturnTime` (clamped), so `isPast(...)` → `isOverdue: true`. PP-1643 states unassigned jobs should never render overdue, but that is a separate ticket, the existing test explicitly asserts this behaviour for this table, and it is unchanged by this PR. Out of PP-1987 scope — noted for awareness only.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ Pre-existing failure | Only `packages/shared/utils/formatWordQuantity.test.ts` fails — a locale artifact (`10,00,000` Indian grouping vs expected `1,000,000`), unrelated to this PR. creative-portal suite: **1684/1684 pass**. |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces (creative-portal included). |
| `npx turbo run lint` | ⚠️ Pre-existing failure | Only `JobReturnTimesTray/index.test.tsx` (5 prettier errors) — a file this PR does not touch (confirmed via `git diff origin/develop...HEAD`). Changed files lint clean. |
| `npx turbo run build` | ✅ | creative-portal + shared build clean (`build --filter=@proofed/creative-portal`, 2/2 tasks). |

Both failures are pre-existing on `develop` and live in files untouched by this PR; per PR scope discipline they are out of scope for this hotfix.

---

## Tests

- ✅ Test added for the projected-window case (unassigned, within hard stop) with deterministic fake timers.
- ✅ Test added for the clamp-to-`maxReturnTime` case (window overruns hard stop).
- ✅ Assigned-job (`returnTime`) test retained and passing.
- ✅ Legacy/no-timing-fields guard tests still pass (no "Invalid Date"/crash).
- ✅ The old assertion that encoded the buggy `maxReturnTime` fallback was removed (not just left alongside the new one).
- ✅ Full creative-portal suite green: 1684 passed / 180 files.
- ⚠️ No automated coverage of the end-to-end accept → display transition (relies on manual/QA per Issue 3).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes root cause (wrong field-selection helper) |
| Regression risk | ✅ Low (one mapper; assigned path unchanged; no user-facing sort) |
| Tests | ✅ New + updated, deterministic; suite green |
| Code quality | ✅ Both nits fixed in `de4f62591` (comment restored; `getEffectiveDeadline` annotated) |
| Validation suite | ⚠️ 2 pre-existing failures in untouched files (locale test + unrelated lint); typecheck + build clean |
| Mergeable state | ✅ Clean (`mergeable_state: clean`; failures are pre-existing on develop) |

---

## Recommendation

**Approve with suggestions**

1. ✅ Done (commit `de4f62591`) — restored the trimmed comment above `isOverdue` in `utils.ts` (Issue 1).
2. ✅ Done (commit `de4f62591`) — annotated `getEffectiveDeadline` as reserved for the PP-1644 admin order-table sort (Issue 2).
3. ⏳ Complete the manual/visual check (unassigned job now shows the estimated user due date) and tick the PR's "Manual testing" / "UI matches designs" boxes. The PR already includes before/after screenshots.
4. ⏳ Confirm with QA/backend that OMS persists the accept-time `returnTime` (Issue 3) so the "before == after" guarantee holds server-side.

The two validation failures are **not blockers** — both are pre-existing on `develop` in files this PR does not modify (a locale-dependent test in `packages/shared` and prettier errors in `JobReturnTimesTray/index.test.tsx`). The PR's own changed files pass typecheck, lint, tests, and build.
