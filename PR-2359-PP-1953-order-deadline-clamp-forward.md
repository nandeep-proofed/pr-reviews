# PR Review: fix/PP-1953: Clamp Workflow Step order deadline forward to last job Job Due Date

**PR:** https://github.com/Proofed/B2BWebserver/pull/2359
**Jira:** https://proofed.atlassian.net/browse/PP-1953
**Status:** In Progress
**Head:** `fix/PP-1953-slide-order-deadline` @ `309580eb` → `develop`
**Files:** 2 changed (+233 / −35)

---

## Jira Requirements vs Implementation

> ⚠️ The ticket's original **Expected Result** ("order deadline recalculated minute-by-minute so the buffer is *preserved*") was **superseded by the client mid-ticket** — Orlin Bonev, comment 62300 (2026-06-25 14:00): *"the order deadline should only be ticking up when the buffer reaches zero. When there is buffer left, we should leave the order deadline as is and just subtract from the buffer."* The PR correctly implements the **client comment**, not the literal original wording.

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Order feasible at setup start must stay submittable — no `Final job maxReturnTime exceeds order dueDateTime` | Deadline clamped forward to last job's Job Due Date once chain overtakes it; buffer floors at 0 so submit validation (`calculateJobsReturnTime`) passes | ✅ Addressed |
| (Client 62300) While Buffer > 0, leave deadline as-is and just subtract from Buffer | `nextDeadline = order.deadline` when not overtaken → early-return, no write; `buffers` memo shrinks each minute | ✅ Addressed |
| (Client 62300) Deadline only ticks up when Buffer reaches 0 | Clamp branch `if (lastJobDueDateUtc && nextDeadline < lastJobDueDateUtc)` fires only when chain ≥ deadline | ✅ Addressed |
| (Original ER) Buffer *preserved* minute-by-minute | Intentionally **not** implemented — replaced by clamp-at-zero per client comment 62300 | ⚠️ Superseded (by design) |
| Missing/past deadline reset (PP-1941 carryover) | `isMissingOrPast` resets to delivery-calc `deadline[0]` / `referenceNowUtc` before clamp | ✅ Preserved |
| Buffer/Deadline clock in phase with Job Due Date chain | `getZonedMinuteNow` floors to the minute; interval aligned to wall-clock `:00` boundary | ✅ Addressed |
| (Comment 62267) "Do we need to fix this in the customer portal as well?" | Not addressed in code — confirmed creative-portal only (see Recommendation) | ✅ Answered |

---

## Architecture Analysis

The fix is localized to `useDeadlineManagement.ts` (creative-portal Workflow step) and changes two things:

1. **Minute clock (`getZonedMinuteNow` + aligned interval).** `todaysDate` is now floored to the minute and the refresh interval is phase-aligned to the wall-clock `:00` boundary (a `setTimeout` to the next minute, then a `setInterval`). This matches the Job Due Date chain's `useZonedTime` clock so the Buffer/Deadline don't lag the displayed chain by 15–30s.

2. **Deadline-reset effect → deadline clamp effect.** The previous "reset only if missing/past" logic is extended: after the missing/past reset, `nextDeadline` is clamped forward to the last non-skipped job's Job Due Date (recovered to UTC via `getReferenceNowUtc`) whenever the chain has overtaken it. A `getTime()` equality early-return keeps the `formik`-dependent effect from looping.

The approach is consistent with the existing PP-1641/PP-1941 machinery (reuses `lastJobMaxReturnTime`, `getReferenceNowUtc`, the same zoned/UTC frame discipline). The displayed Buffer is driven by the live `currentBuffers` memo (confirmed: `BufferSection` renders `getFormattedBuffer(currentBuffers)`), so the "subtract from the buffer" behaviour is actually visible. Submit-side validation is unchanged and is satisfied by construction because the clamp floors the buffer at 0.

Regression surface is low: the hook's return shape is unchanged, and consumers (`WorkflowStep/index.tsx`, `BufferSection`, `WorkflowWindowInputRow`) read the same fields.

---

## Issues Found

### 1. `currentBuffers` sort is lexicographic and mutates the memo — Pre-existing (out of scope, not introduced by this PR)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** `currentBuffers` memo

**Severity:** low

**Problem:** `[...new Set(buffers.sort())]` calls `Array.prototype.sort()` with no comparator (lexicographic string order, e.g. `[5, 120]` → `[120, 5]`) and mutates the memoized `buffers` array in place. `currentBuffers[0]`/`[1]` feed `getFormattedBuffer` (range display) and `handleUpdateBuffer`.

**Impact:** With 2+ distinct buffer values across selected orders, the displayed Buffer range can render reversed and `handleUpdateBuffer` can pick the wrong "first" buffer. Not triggered for single-order flows.

**Note:** This line is **not part of this PR's diff** — it is pre-existing on `develop`. Out of scope for PP-1953; flagging for a separate follow-up, not a blocker.

**Fix:**

```typescript
const currentBuffers = useMemo(
  () => [...new Set(buffers)].sort((a, b) => a - b),
  [buffers]
);
```

### 2. Explanatory comments dropped from the modified effects — Intentional (per author preference)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** minute-interval effect & deadline-clamp effect

**Severity:** low

**Problem:** The diff removed the `develop` comments — notably the PP-1941 block explaining *why* `getReferenceNowUtc` is needed (canonical-UTC vs zoned `todaysDate` comparison) and the "Update orders deadline if not set or invalid" header. The new clamp logic ships with no header comment describing the hold-then-clamp semantics.

**Impact:** The timezone-frame reasoning (a genuine footgun this code already tripped over in PP-1858/PP-1941) is no longer documented inline, raising the chance a future edit reintroduces a double-offset or past-deadline-skew bug.

**Disposition:** Intentional — the author chose to ship without added comments this round. Non-blocking; can be revisited if the preference changes.

**Fix (if revisited):** Re-add a short header on the clamp effect and retain the PP-1941 canonical-UTC rationale.

### 3. `sharedOrder.deadline` sync uses a stale read inside the loop — Pre-existing (out of scope, not introduced by this PR)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** deadline-clamp effect

**Severity:** low

**Problem:** Inside `formik.values.orders.forEach(...)`, the shared-deadline bump compares against `formik.values.sharedOrder.deadline`, which does not update between synchronous iterations. With multiple orders this is last-write-wins rather than a guaranteed max.

**Impact:** Minor; only multi-order setups with differing clamped deadlines. The `getTime()` early-return means the steady (buffer > 0) state never reaches this block, so no churn. Pre-existing behaviour, not introduced here.

**Fix:** If tightened later, compute the max clamped deadline across orders once, then write `sharedOrder.deadline` after the loop.

### 4. Branch name no longer matches the implementation — Intentional (kept on the same branch by request)

**[File: n/a — repo metadata]**

**Severity:** low

**Problem:** Branch is `fix/PP-1953-slide-order-deadline`, from the earlier (discarded) per-minute-slide approach. The shipped behaviour is clamp-at-zero.

**Impact:** Cosmetic/traceability only; called out in the PR body already.

**Disposition:** Intentional — work continued on the same branch by request; not renaming.

**Fix:** None needed — the PR squash-merges and the PR title is accurate.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | Touched suite 9/9 pass (4 PP-1641 + 2 PP-1941 + 3 new PP-1953). Full run: 1 **pre-existing, unrelated** failure in `@proofed/shared/utils/formatWordQuantity.test.ts` (`"10,00,000"` Indian locale grouping vs `"1,000,000"`) — no files touched in `packages/shared`. |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces. |
| `npx turbo run lint` | ✅ | Clean on touched files (one prettier wrap auto-fixed; pre-commit hook re-ran clean). |
| `npx turbo run build` | ✅ | 4/4 tasks successful (exit 0); creative-portal build clean. |

---

## Tests

- ✅ New `deadline clamp-forward (PP-1953)` unit suite added (3 tests): clamp once chain overtakes (buffer floors at 0), positive-buffer deadline untouched, minute-floor anchoring (no sub-minute drift).
- ✅ Existing PP-1641 buffer-memo and PP-1941 reset suites still pass (no regression).
- ✅ Meets the "every PR includes tests for new code" requirement.
- ⚠️ No automated coverage that submit (`validateOrderJobTimings` / `calculateJobsReturnTime`) actually succeeds end-to-end after a clamp — covered indirectly (buffer floors at 0) but a small integration-style assertion would harden it. Optional.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Implements the agreed client behaviour (clamp-at-zero) and fixes the rejection |
| Regression risk | ✅ Low — localized to one hook, return shape unchanged |
| Tests | ✅ New suite + existing suites pass |
| Code quality | ✅ Consistent with existing patterns (minor: dropped comments, pre-existing sort) |
| Validation suite | ✅ All pass (test / typecheck / lint / build) |
| Mergeable state | ✅ Clean (GitHub) |

---

## Recommendation

**Approve with suggestions.**

1. **Validation green** — `test`, `typecheck`, `lint`, and `build` (4/4 tasks, exit 0) all pass. The one `turbo run test` failure is a pre-existing, unrelated locale issue in `@proofed/shared` (`formatWordQuantity`), not touched by this PR.
2. **Customer-portal question (comment 62267) — answered: creative-portal only.** Investigation confirmed customer-portal order creation is single-shot and computes job timings fresh at submit (`calculateJobsReturnTime` with `orderStartTime = new Date()`), with the minute refetch being display-only. No frozen-deadline-vs-sliding-chain asymmetry exists there, so no customer-portal fix is required.
3. **Issues 1 & 3 — pre-existing, out of scope.** Not introduced by this PR; leave for PP-1953 and file separately if desired (per scope discipline).
4. **Issues 2 & 4 — intentional.** Comments omitted by author preference (Issue 2) and the branch name kept on purpose (Issue 4); neither blocks merge.
