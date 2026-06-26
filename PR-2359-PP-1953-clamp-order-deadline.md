# PR Review: fix/PP-1953: Clamp Workflow Step order deadline forward to last job Job Due Date

**PR:** https://github.com/Proofed/B2BWebserver/pull/2359
**Jira:** https://proofed.atlassian.net/browse/PP-1953
**Status:** In Progress (Bug, High priority)

---

## Jira Requirements vs Implementation

The ticket's written **Expected Result** ("recalculate the order deadline minute-by-minute so the buffer is *preserved*") was **superseded on the call** — Jira comment 62300 (Orlin) and confirmed in 62333 (Nandeep): *"the order deadline should only be ticking up when the buffer reaches zero. When there is buffer left, leave the deadline as is and just subtract from the buffer."* The PR correctly implements the **agreed clamp-at-zero** behaviour, not the stale ticket text.

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| A feasible-at-start order must stay submittable; submit no longer blocked by `Final job maxReturnTime exceeds order dueDateTime` | The minute-tick effect clamps `orders[i].deadline` forward to the last job's Job Due Date once the chain overtakes it, so the client check (`validateOrderJobTimings` → `calculateJobsReturnTime`) and the server mirror both pass | ✅ Addressed |
| Agreed behaviour: Buffer > 0 → deadline held, only buffer counts down | When `lastJobDueDateUtc < nextDeadline` no clamp occurs and the early-return (`nextDeadline === order.deadline`) leaves the deadline untouched; `currentBuffers` memo recomputes each minute | ✅ Addressed |
| Agreed behaviour: Buffer = 0 → deadline ticks up to last Job Due Date; buffer floors at 0 | Clamp sets `nextDeadline = lastJobDueDateUtc` (= minute-floored now + Σ maxReturnWindowsMinutes); buffer then computes to exactly 0 | ✅ Addressed |
| Stay in phase with the per-job Job Due Date chain (no sub-minute lag) | `todaysDate` is now minute-floored (`getZonedMinuteNow`) and the tick is aligned to the wall-clock `:00` boundary (setTimeout → setInterval), matching `useZonedTime`'s minute quantization used by the display chain | ✅ Addressed |
| Preserve PP-1941 (missing/past deadline reset to delivery-calc value, UTC-frame comparison) | `getReferenceNowUtc(todaysDate, timeZone)` retained; `isMissingOrPast` resets to `fallbackDeadline` before the clamp | ✅ Addressed |
| Every PR includes tests for new code | New `deadline clamp-forward (PP-1953)` suite — 3 cases (clamp + floor, hold-with-buffer, minute-floor anchoring) | ⚠️ Partial — covers the on-mount clamp math but not the minute-tick progression or the new alignment timer (see Issue 5) |
| Customer Portal: does it need the same fix? (open question, cmt 62267) | Not addressed; PR is Creative Portal only | ⚠️ Open — unanswered in Jira; confirm whether CP is in scope as a follow-up |

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

### 2. Buffer is not floored at 0 in the memo — possible one-frame negative flash at the overtake tick

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** useDeadlineManagement — `buffers` / `currentBuffers` memo

**Severity:** low

**Problem:** `buffers` returns `differenceInMinutes(zonedDeadline, lastMax ?? todaysDate)` with no `Math.max(0, …)`. The clamp that guarantees `deadline ≥ chainEnd` runs in an effect (after paint), while the memo computes during render. On the exact minute the chain overtakes the deadline, the memo can compute a negative value (truncated, so at most `-1`) for one painted frame before the clamp effect commits and re-renders to `0`.

**Impact:** A brief flicker of a negative/`-1` buffer in the Buffer pill at the moment of overtake. Cosmetic and sub-second; tests don't catch it because they assert only after `await act` settles. The PR's stated intent is "Buffer floors at 0".

**Fix:** Floor the per-order buffer in the memo so the displayed value can never dip below zero regardless of effect timing:

```typescript
return Math.max(
  0,
  differenceInMinutes(zonedDeadline, lastMax ?? todaysDate)
);
```

### 3. `getZonedMinuteNow` duplicates `useZonedTime`'s minute-quantization instead of reusing it

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** getZonedMinuteNow

**Severity:** low

**Problem:** `getZonedMinuteNow` re-implements exactly what `apps/creative-portal/hooks/useZonedTime.ts` already does — `toZonedTime(new Date(Math.floor(Date.now()/60_000)*60_000), timeZone)`. The display chain (`useWorkflowWindowInputRow`) gets its clock from `useZonedTime`; this hook now maintains a second, formula-duplicated clock. Phase alignment between the clamped deadline and the displayed Job Due Date depends on the two formulas staying byte-identical.

**Impact:** No current defect — the formulas match, so both land on the same minute. But it's a reuse-first violation (per CLAUDE.md) and a latent drift risk: if `useZonedTime`'s quantization is ever changed, the deadline would silently fall out of phase with the display chain again (the exact class of bug this ticket fixes). `useDeadlineManagement` still needs its own `setInterval` to *drive* re-renders (a memo-only `useZonedTime` doesn't tick), but it could source the `todaysDate` *value* from `useZonedTime()` and use the interval purely as a render trigger.

**Fix:** Consider deriving `todaysDate` from `useZonedTime()` and using the aligned interval only to force a re-render (e.g. a tick counter), so a single quantization definition feeds both clocks. At minimum, add a comment cross-referencing `useZonedTime` so the two stay in sync.

### 4. Residual submit race at the minute boundary

**[File: apps/creative-portal/components/organisms/NewOrderForm/index.tsx]**

**Function/Class:** onSubmit → validateOrderJobTimings

**Severity:** low

**Problem:** The submit check passes `orderStartTime: new Date()` (live, sub-minute) and chains from it, while the clamped `order.deadline` is anchored on the *minute-floored* now. Within the same wall-clock minute the `startOfMinute(finalMaxReturn) > startOfMinute(dueDate)` truncation collapses the two, so the check passes. But in the brief window after the wall clock crosses a minute boundary and before the re-clamp effect commits the new (later) deadline, a zero-buffer order submitted at that instant can still throw `Final job maxReturnTime exceeds order dueDateTime`.

**Impact:** A theoretical sub-second failure window at minute rollover for an order sitting at exactly zero buffer. It self-heals on the next render/tick (unlike the pre-fix frozen deadline, which never recovered), so this is a massive net improvement; flagging for completeness rather than as a defect. If fully deterministic submit is desired, anchoring `orderStartTime` to the same minute-floored now used for the clamp (instead of `new Date()`) would close it.

### 5. New clock/progression code is not directly tested

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.test.ts]**

**Function/Class:** deadline clamp-forward (PP-1953) suite

**Severity:** low

**Problem:** All three new tests assert the **on-mount** clamp (deadline already overtaken when the hook first renders). None advance fake timers (`vi.advanceTimersByTime(60_000)`), so neither the minute-tick clamp progression (the actual elapsed-time bug scenario) nor the new wall-clock alignment (`setTimeout` → `setInterval`) is exercised. The `useEffect` timer block has no coverage.

**Impact:** The core clamp math is well covered, but a regression in the tick/alignment wiring (e.g. wrong `msUntilNextMinute`, missing `clearTimeout`, interval not re-armed) would not be caught.

**Fix:** Add a case that mounts with a positive buffer, advances fake timers across one or more minute boundaries, and asserts the deadline holds while buffer > 0 then clamps forward once the chain overtakes — and that the buffer counts down across ticks. This also documents the intended minute-by-minute behaviour as executable spec.

### 6. Lost rationale comment / branch-name mismatch (informational)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**

**Function/Class:** clamp effect

**Severity:** low

**Problem:** The PP-1941 explanatory comment above the old effect (why `getReferenceNowUtc` recovers UTC before the past-deadline comparison) was removed; the logic is preserved but the rationale is gone. Separately, the branch is named `…slide-order-deadline` from the earlier exploratory approach while the implemented behaviour is clamp-at-zero (acknowledged in the PR description).

**Impact:** Minor maintainability/clarity. No functional effect.

**Fix:** Re-add a short comment summarising the UTC-recovery + clamp intent. Branch name is cosmetic; squash-merge subject should reflect "clamp", which it does.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped — user opted out | Working tree on PP-1890 with uncommitted changes; validation would require a fresh worktree + install. Not run. |
| `npx turbo run typecheck` | ⏭️ Skipped — user opted out | PR description claims `turbo run typecheck` green; not independently verified. |
| `npx turbo run lint` | ⏭️ Skipped — user opted out | PR description claims lint clean on touched files; not independently verified. |
| `npx turbo run build` | ⏭️ Skipped — user opted out | Not run. |

---

## Tests

- ✅ New `deadline clamp-forward (PP-1953)` suite added (3 cases): clamp-forward + buffer floors at 0; positive-buffer hold; minute-floor anchoring (no sub-minute drift).
- ✅ Existing PP-1641 buffer-memo and PP-1941 reset suites retained and consistent with the refactor.
- ⚠️ No fake-timer progression test for the minute-tick clamp or the wall-clock alignment timer (Issue 5).
- ⏭️ Suite not executed this review (validation skipped per user). Per CLAUDE.md, `npx turbo run test` must pass with 0 failures before merge — **re-run before merging.**

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fix matches the on-call agreed behaviour and resolves the client + server rejection by construction |
| Regression risk | ✅ Low — change localized to one hook; public API unchanged; single consumer |
| Tests | ⚠️ Core clamp covered; minute-tick progression + alignment timer untested |
| Code quality | ⚠️ Good; minor reuse (Issue 3), buffer-floor (Issue 2), and multi-order sync (Issue 1) refinements |
| Validation suite | ⏭️ Skipped — user opted out; re-run before merge |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`); validation not independently confirmed |

---

## Recommendation

**Approve with suggestions** (conditional on validation being re-run).

1. **Re-run the mandatory validation suite** (`test` / `typecheck` / `lint` / `build`) on the PR branch before merging — it was skipped in this review, and CLAUDE.md forbids merging on any failure. The PR claims typecheck/lint are green but build and the full test run were not independently confirmed here.
2. **(Issue 1)** Compute `sharedOrder.deadline` once (max over orders) and write it after the loop, instead of reading stale Formik state per iteration.
3. **(Issue 2)** Floor the `buffers` memo at 0 so a one-frame negative buffer can't flash at the overtake tick.
4. **(Issue 5)** Add a fake-timer test that advances across minute boundaries to lock in the hold-then-clamp progression and exercise the new alignment timer.
5. **(Issue 3, nice-to-have)** Reuse `useZonedTime`'s quantization (or cross-reference it) so the deadline clock can't drift out of phase with the display chain later.
6. **Process:** Resolve the open Jira question (cmt 62267 — does the Customer Portal need the same fix?) and update the ticket's stale "Expected Result" text to reflect the agreed clamp-at-zero behaviour.

None of the findings are blockers on their own; the fix is sound and root-caused. The only hard gate is re-running validation per project policy.
