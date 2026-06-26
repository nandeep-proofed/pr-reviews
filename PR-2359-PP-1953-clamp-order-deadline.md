# PR Review: fix/PP-1953: Clamp Workflow Step order deadline forward to last job Job Due Date

**PR:** https://github.com/Proofed/B2BWebserver/pull/2359
**Jira:** https://proofed.atlassian.net/browse/PP-1953
**Status:** In Progress (Bug, High priority)

> **Update (follow-up commit `6c7c30d8`):** Issues 2 and 5 below have been **resolved** — the `buffers` memo is now floored at 0, and a fake-timer minute-tick progression test was added. Sections updated in place; resolved items are marked **✅ Resolved**.

---

## Jira Requirements vs Implementation

The ticket's written **Expected Result** ("recalculate the order deadline minute-by-minute so the buffer is *preserved*") was **superseded on the call** — Jira comment 62300 (Orlin) and confirmed in 62333 (Nandeep): *"the order deadline should only be ticking up when the buffer reaches zero. When there is buffer left, leave the deadline as is and just subtract from the buffer."* The PR correctly implements the **agreed clamp-at-zero** behaviour, not the stale ticket text.

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| A feasible-at-start order must stay submittable; submit no longer blocked by `Final job maxReturnTime exceeds order dueDateTime` | The minute-tick effect clamps `orders[i].deadline` forward to the last job's Job Due Date once the chain overtakes it, so the client check (`validateOrderJobTimings` → `calculateJobsReturnTime`) and the server mirror both pass | ✅ Addressed |
| Agreed behaviour: Buffer > 0 → deadline held, only buffer counts down | When `lastJobDueDateUtc < nextDeadline` no clamp occurs and the early-return (`nextDeadline === order.deadline`) leaves the deadline untouched; `currentBuffers` memo recomputes each minute | ✅ Addressed |
| Agreed behaviour: Buffer = 0 → deadline ticks up to last Job Due Date; buffer floors at 0 | Clamp sets `nextDeadline = lastJobDueDateUtc` (= minute-floored now + Σ maxReturnWindowsMinutes); buffer memo now floors at 0 (`Math.max(0, …)`) so it cannot flash negative | ✅ Addressed |
| Stay in phase with the per-job Job Due Date chain (no sub-minute lag) | `todaysDate` is now minute-floored (`getZonedMinuteNow`) and the tick is aligned to the wall-clock `:00` boundary (setTimeout → setInterval), matching `useZonedTime`'s minute quantization used by the display chain | ✅ Addressed |
| Preserve PP-1941 (missing/past deadline reset to delivery-calc value, UTC-frame comparison) | `getReferenceNowUtc(todaysDate, timeZone)` retained; `isMissingOrPast` resets to `fallbackDeadline` before the clamp | ✅ Addressed |
| Every PR includes tests for new code | `deadline clamp-forward (PP-1953)` suite — now 4 cases incl. a fake-timer minute-tick progression test (Buffer counts down 2→1→0, deadline held, then clamps forward) covering the elapsed-time scenario and the alignment timer | ✅ Addressed |
| Customer Portal: does it need the same fix? (open question, cmt 62267) | Not addressed; PR is Creative Portal only — investigation confirmed CP order creation is single-shot and computes job timings fresh at submit, so it has no equivalent drift | ✅ Answered (creative-portal only) |

---

## Architecture Analysis

The bug is an asymmetry introduced by Dynamic Deadlines (PP-1641): the per-job Job Due Date chain is anchored on a live, minute-ticking "now" (`useWorkflowWindowInputRow` via `useZonedTime`), so it slides forward every minute, while the order deadline was initialized once and frozen. Buffer = `Deadline − lastJobMaxReturnTime` therefore drifts negative as setup time elapses, and both the client mirror check (`validateOrderJobTimings`) and the server check (`createOrderBusinessLogic` via the shared `calculateJobsReturnTime`) reject with `Final job maxReturnTime exceeds order dueDateTime`.

The fix reworks the existing "update deadline if missing/invalid" effect in `useDeadlineManagement.ts` into a clamp:

- `nextDeadline` starts from the current `order.deadline` (or the delivery-calc fallback if missing/past, preserving PP-1941).
- If `nextDeadline < lastJobDueDateUtc` (the chain has overtaken it), `nextDeadline` is pulled forward to the chain end.
- An early-return skips the Formik write when nothing changed, preventing setstate churn.

Two supporting changes make the clamp land on the *same* minute the display chain shows:

1. `getZonedMinuteNow()` floors `Date.now()` to the minute before zoning — identical to `useZonedTime`'s `Math.floor(Date.now()/60_000)*60_000` quantization.
2. The tick is re-armed to fire on the wall-clock `:00` boundary (setTimeout to the next minute, then setInterval).

This is correct because the submit-time check truncates to minute precision (`startOfMinute(finalMaxReturn) > startOfMinute(dueDate)`). The clamped deadline is `flooredNow + Σ`; the submit chains from `new Date()` (`liveNow + Σ`); both share the same wall-clock minute, so `startOfMinute` collapses them and the check passes. The approach reuses the shared `calculateJobsReturnTime` for client/server agreement, so a clamped deadline that satisfies the client mirror satisfies the server by construction. The change is localized to one hook; the only public surface (`currentBuffers`, `todaysDate`, handlers) is unchanged, so blast radius is limited to the Workflow step. Sole consumer is `WorkflowStep/index.tsx`.

---

## Issues Found

### 1. Multi-order `sharedOrder.deadline` sync relies on stale Formik reads within the loop

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** useDeadlineManagement — clamp effect

**Severity:** low

**Problem:** The effect loops over `formik.values.orders` and, per order, conditionally raises `sharedOrder.deadline` to that order's `nextDeadline`. `formik.setFieldValue` is batched, so within a single synchronous pass each iteration reads the *original* `sharedOrder.deadline`, not the value a previous iteration just wrote. With multiple selected orders that clamp to different deadlines, the loop is last-writer-wins, not max-wins, on the first render.

**Impact:** For a multi-order batch with differing clamped deadlines, `sharedOrder.deadline` can momentarily reflect a smaller order's deadline rather than the maximum. Because `formik` identity changes every render and is in the dep array, the effect re-runs and the value converges to the true max over a few renders — so this is self-correcting and not a user-visible defect in practice, but the per-iteration logic is fragile and would break if the early-return or deps were changed later.

**Disposition:** Pre-existing pattern, self-correcting; left as an optional follow-up (not in this PR's scope).

**Fix:** Compute the intended shared deadline once from the order set (e.g. `Math.max` of all clamped `nextDeadline`s) and write it a single time after the loop, rather than reading `formik.values.sharedOrder.deadline` inside the loop:

```typescript
let sharedNext = formik.values.sharedOrder?.deadline;
formik.values.orders.forEach((order, index) => {
  // ...compute nextDeadline, write orders[index].deadline...
  if (!sharedNext || sharedNext < nextDeadline) sharedNext = nextDeadline;
});
if (sharedNext !== formik.values.sharedOrder?.deadline) {
  formik.setFieldValue("sharedOrder.deadline", sharedNext);
}
```

### 2. Buffer is not floored at 0 in the memo — possible one-frame negative flash at the overtake tick — ✅ Resolved (commit `6c7c30d8`)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** useDeadlineManagement — `buffers` / `currentBuffers` memo

**Severity:** low

**Problem:** `buffers` returned `differenceInMinutes(zonedDeadline, lastMax ?? todaysDate)` with no `Math.max(0, …)`. The clamp that guarantees `deadline ≥ chainEnd` runs in an effect (after paint), while the memo computes during render. On the exact minute the chain overtakes the deadline, the memo could compute a negative value (truncated, so at most `-1`) for one painted frame before the clamp effect commits and re-renders to `0`.

**Impact:** A brief flicker of a negative/`-1` buffer in the Buffer pill at the moment of overtake. Cosmetic and sub-second.

**Resolution:** The memo now returns `Math.max(0, differenceInMinutes(...))`, so the displayed Buffer can never dip below zero regardless of effect timing. The new minute-tick progression test (Issue 5) asserts the Buffer reads `0` — not `-1` — at the overtake minute.

### 3. `getZonedMinuteNow` duplicates `useZonedTime`'s minute-quantization instead of reusing it

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** getZonedMinuteNow

**Severity:** low

**Problem:** `getZonedMinuteNow` re-implements exactly what `apps/creative-portal/hooks/useZonedTime.ts` already does — `toZonedTime(new Date(Math.floor(Date.now()/60_000)*60_000), timeZone)`. The display chain (`useWorkflowWindowInputRow`) gets its clock from `useZonedTime`; this hook now maintains a second, formula-duplicated clock. Phase alignment between the clamped deadline and the displayed Job Due Date depends on the two formulas staying byte-identical.

**Impact:** No current defect — the formulas match, so both land on the same minute. But it's a reuse-first violation (per CLAUDE.md) and a latent drift risk: if `useZonedTime`'s quantization is ever changed, the deadline would silently fall out of phase with the display chain again (the exact class of bug this ticket fixes). `useDeadlineManagement` still needs its own `setInterval` to *drive* re-renders (a memo-only `useZonedTime` doesn't tick), but it could source the `todaysDate` *value* from `useZonedTime()` and use the interval purely as a render trigger.

**Disposition:** Optional follow-up; the fix for #2's progression test also indirectly guards phase alignment.

**Fix:** Consider deriving `todaysDate` from `useZonedTime()` and using the aligned interval only to force a re-render (e.g. a tick counter), so a single quantization definition feeds both clocks. At minimum, add a comment cross-referencing `useZonedTime` so the two stay in sync.

### 4. Residual submit race at the minute boundary

**[File: apps/creative-portal/components/organisms/NewOrderForm/index.tsx]**

**Function/Class:** onSubmit → validateOrderJobTimings

**Severity:** low

**Problem:** The submit check passes `orderStartTime: new Date()` (live, sub-minute) and chains from it, while the clamped `order.deadline` is anchored on the *minute-floored* now. Within the same wall-clock minute the `startOfMinute(finalMaxReturn) > startOfMinute(dueDate)` truncation collapses the two, so the check passes. But in the brief window after the wall clock crosses a minute boundary and before the re-clamp effect commits the new (later) deadline, a zero-buffer order submitted at that instant can still throw `Final job maxReturnTime exceeds order dueDateTime`.

**Impact:** A theoretical sub-second failure window at minute rollover for an order sitting at exactly zero buffer. It self-heals on the next render/tick (unlike the pre-fix frozen deadline, which never recovered), so this is a massive net improvement; flagging for completeness rather than as a defect. If fully deterministic submit is desired, anchoring `orderStartTime` to the same minute-floored now used for the clamp (instead of `new Date()`) would close it.

**Disposition:** Theoretical, self-healing; documented optional follow-up.

### 5. New clock/progression code is not directly tested — ✅ Resolved (commit `6c7c30d8`)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.test.ts]**

**Function/Class:** deadline clamp-forward (PP-1953) suite

**Severity:** low

**Problem:** The original three tests asserted only the **on-mount** clamp (deadline already overtaken when the hook first renders). None advanced fake timers, so neither the minute-tick clamp progression (the actual elapsed-time bug scenario) nor the new wall-clock alignment (`setTimeout` → `setInterval`) was exercised.

**Resolution:** Added a fake-timer test that mounts with a positive Buffer (2 min) and advances across three minute boundaries (`vi.advanceTimersByTime(60_000)` per tick), asserting: Buffer counts down 2 → 1 → 0 while the deadline is **held**, then the deadline **clamps forward** once the chain overtakes it (at +3 min) with the Buffer floored at 0. This exercises the minute-tick progression and the wall-clock-aligned interval, and doubles as executable spec for the agreed behaviour. Suite is now 10 tests (was 9), all passing.

### 6. Lost rationale comment / branch-name mismatch (informational)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** clamp effect

**Severity:** low

**Problem:** The PP-1941 explanatory comment above the old effect (why `getReferenceNowUtc` recovers UTC before the past-deadline comparison) was removed; the logic is preserved but the rationale is gone. Separately, the branch is named `…slide-order-deadline` from the earlier exploratory approach while the implemented behaviour is clamp-at-zero (acknowledged in the PR description).

**Impact:** Minor maintainability/clarity. No functional effect.

**Disposition:** Intentional — author opted to ship without added comments; branch name kept on the same branch by request.

---

## Validation Checks

Run against the PR branch at follow-up commit `6c7c30d8`.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | Touched suite **10/10** pass (incl. the new minute-tick progression test). Full-repo run on the prior commit had 1 **pre-existing, unrelated** failure in `@proofed/shared/utils/formatWordQuantity.test.ts` (Indian-locale digit grouping) — no files touched in `packages/shared`. |
| `npx turbo run typecheck` | ✅ | creative-portal typecheck clean (0 errors). |
| `npx turbo run lint` | ✅ | Clean on touched files. |
| `npx turbo run build` | ✅ | Full `turbo run build` was green (4/4 tasks) on the prior commit `309580eb`; this follow-up delta is a one-line `Math.max` + a test file, so build is unaffected. |

---

## Tests

- ✅ `deadline clamp-forward (PP-1953)` suite — now **4 cases**: clamp-forward + buffer floors at 0; positive-buffer hold; minute-floor anchoring (no sub-minute drift); **minute-tick progression** (Buffer 2→1→0, deadline held, then clamp forward) covering the elapsed-time scenario and the alignment timer.
- ✅ Existing PP-1641 buffer-memo and PP-1941 reset suites retained and consistent with the refactor.
- ✅ 10/10 tests pass.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fix matches the on-call agreed behaviour and resolves the client + server rejection by construction |
| Regression risk | ✅ Low — change localized to one hook; public API unchanged; single consumer |
| Tests | ✅ Clamp math + minute-tick progression + alignment timer all covered (10/10) |
| Code quality | ✅ Good; minor optional refinements remain (Issues 1, 3, 4) |
| Validation suite | ✅ test / typecheck / lint / build all green |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Approve.**

1. **Issues 2 and 5 resolved** in follow-up commit `6c7c30d8` — Buffer memo floored at 0; minute-tick progression test added. Validation (test / typecheck / lint / build) all green.
2. **Customer Portal (cmt 62267) — answered: creative-portal only.** CP order creation is single-shot and computes job timings fresh at submit, so it has no equivalent frozen-deadline-vs-sliding-chain drift; no CP fix required.
3. **Optional follow-ups (non-blocking):** Issue 1 (compute `sharedOrder.deadline` max-after-loop), Issue 3 (reuse `useZonedTime` quantization / cross-reference), Issue 4 (anchor `orderStartTime` to minute-floored now for a fully deterministic submit). None are required for this fix.
4. **Process:** update the ticket's stale "Expected Result" text to reflect the agreed clamp-at-zero behaviour.
