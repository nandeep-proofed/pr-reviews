# PR Review: fix/PP-2020: stop refusing a QA skip that would only tighten the schedule

**PR:** https://github.com/Proofed/B2BWebserver/pull/2431

**Jira:** https://proofed.atlassian.net/browse/PP-2020

**Status:** Code Review

**Author:** nandeep-proofed · **Branch:** `fix/PP-2020-skip-blocked-by-committed-deadline` → `develop` · **Head:** `ec5b1f4`

**Diff:** 2 files, +66 / −23 (both in `@proofed/creative-portal`)

---

## What this means for users (non-technical summary)

1. **The reported problem is genuinely fixed for the shape it was reported in.** When the last job in an order is already booked out to someone and has no spare time, an admin can now skip an earlier job instead of being blocked. Verified by running the changed code.
2. **The same block still happens when the booked-out job is in the middle of the order, not at the end** — and the message an admin sees in that case is now *less* helpful than the one it replaced. It says the order's dates are out of order, which sounds like corrupt data and gives no hint that the documented workaround (push the deadlines out first) applies.
3. **In one situation a skip can now push a job's cut-off time *later* than it was.** Skipping is supposed to only ever free up time. This only bites on orders where an admin previously moved a job's user due date past its own cut-off using the sidebar picker — which the sidebar currently permits for downstream jobs.
4. **No automated test now covers any situation where a skip is refused.** The change deleted the only such test and did not replace it, at the same moment it made that refusal path the one that matters.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| Skipping a Review/QA job "shifts subsequent jobs' deadlines proportionately" | `planJobRemovalTiming` still collapses every surviving downstream `maxReturnTime` by the removed span; unchanged behaviour for unassigned downstream jobs | ✅ Addressed |
| "…and allows the skip when there is sufficient buffer" | The `returnTimeExceedsMax`-style refusal is replaced by a clamp at the committed `returnTime` — the skip proceeds | ⚠️ Partial — holds only when the clamped job has no surviving job after it (see Issue 1) |
| Root cause identified (intermittent / "similar orders skipped QA fine") | Correctly diagnosed: the check only fires when `returnTime` is set, which requires assignment. The PR's inequality and the "assigned vs unassigned" explanation match the code | ✅ Addressed |
| Reproduce against reported order 53159 | Reproduced on B2B test order 21369 instead; the PR itself flags 53159 as unconfirmed | ⚠️ Partial — author already called this out for the PO |

**Scope:** tight and single-purpose. No scope creep — the diff touches only the planner and its test file. Title and branch follow `fix/PP-XXXX: <subject>`.

---

## Architecture Analysis

`planJobRemovalTiming` is a pure planner: given the order's pre-delete jobs and the ids being removed, it returns the net `maxReturnTime` changes for surviving downstream jobs. Two callers consume it, both planning **before** any write so a refusal leaves the order untouched:

- `components/organisms/sidebars/contents/OrderManagment/hooks.tsx:324` — single-order skip from the sidebar
- `contexts/bulkOrderActionsContext/provider.tsx:628` — bulk skip, per-order, blocked orders reported and the rest proceed

The change swaps a hard refusal for a `Math.max` clamp at the assigned job's committed `returnTime`, and suppresses the write when the clamp is total. The reasoning in the PR description is sound and the diagnosis is correct — this is a well-argued change with an honest "for review" section.

The gap is that clamping is applied **per job** while the ordering invariant it breaks is **chain-wide**. The strictly-increasing backstop at line 139 catches the resulting inconsistency and refuses the whole skip, so nothing corrupt is ever written — but that means the fix is complete only for the case where the clamped job is last in the chain. The PR's own updated comment acknowledges the backstop "now guards a real case"; what it does not say is that the real case it guards is the very bug being fixed, one position earlier in the chain.

**Verification method:** the PR's two source files were checked out into a clean working tree and the planner was exercised directly with `vitest`. The module's 12 tests pass, and two additional probe cases (removed afterwards) produced the evidence quoted in Issues 1 and 2.

---

## Issues Found

### 1. The clamp only rescues the last job in the chain — a mid-chain assigned job still refuses the skip, with a worse message

**[File: apps/creative-portal/api/utils/jobs/planJobRemovalTiming.ts]**

> **In plain terms:** The fix works when the booked-out job is the final job on the order — which is the case that was reported. If a booked-out job sits in the middle instead, the admin is still blocked from skipping, and the message they get now says the order's dates are in the wrong order. That reads like a data fault, tells them nothing they can act on, and no longer points at the workaround the ticket records ("manually shift the deadlines forward, then skip"). The admin is worse off than before for this shape, because the old message at least named the job and the reason.

**Function/Class:** `planJobRemovalTiming`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Log in as an Admin and open an order whose chain is Editing → Review → QA → Return, where **QA is assigned** and its committed deadline sits at or near its own hard stop (the PP-2020 condition), and Return sits shortly after QA.
2. Skip the **Review** job — either from the order side panel, or from the orders table via **More → Skip** on the current job.
3. **Expected:** per the PR's own principle ("skipping frees time, so it can never *need* to tighten the schedule"), the skip succeeds; QA holds at its committed deadline and Return collapses.
4. **Actual:** the skip is refused with the toast *"Skipping would break job due-date ordering."*

**Problem:** The clamp is applied independently per job, but the ordering invariant it violates is chain-wide. Once an upstream survivor is held at its `returnTime`, the next survivor is still collapsed by the full `cumulativeRemovedMs` and lands **before** it, so the strictly-increasing backstop rejects the entire operation.

**Evidence:** `planJobRemovalTiming.ts:114-117` clamps, and `planJobRemovalTiming.ts:139-148` then refuses:

```typescript
const newMaxMs =
  node.returnTimeMs != null
    ? Math.max(collapsedMs, node.returnTimeMs)
    : collapsedMs;

survivingMaxes.push(newMaxMs);
// ...
const isStrictlyIncreasing = survivingMaxes.every(
  (value, index) => index === 0 || value > survivingMaxes[index - 1]
);
```

Executed against the PR's source with a chain of Editing `+1h` → Review `+10h` (removed) → QA `+11h` (`status: "Assigned"`, `returnTime: +11h`) → Return `+11h20m`:

```
{"ok":false,"reason":"Skipping would break job due-date ordering."}
```

QA clamps to `+11h`; Return collapses to `680m − 540m = +2h20m`, which is earlier than QA — hence the refusal. The condition is `removedSpan > (nextJob.max − clampedJob.max)`, i.e. the removed job's window exceeds the gap to the following job. With Review windows measured in hours and QA→Return gaps in tens of minutes, that inequality is the normal case rather than an exotic one, so **whenever a clamped job has a surviving job after it, this will essentially always trip.**

Reachability of a ≥2-survivor chain: the bulk **Skip current job** action passes `currentJobIds` with no Review+QA pairing (`components/molecules/tables/TableWithFilters/utils/getBulkActionsConfig.tsx:196-208`, gated by `isAbleToSkipCurrentJob` = every current job has `allowDeletion`, `contexts/bulkOrderActionsContext/provider.tsx:453-457`). Skipping a current Review there leaves QA **and** Return downstream. The sidebar's Review+QA pairing (`OrderManagment/hooks.tsx:312-318`) avoids this for the specific Review case, which is why the reported order 21369 is fixed.

**Impact:** PP-2020's acceptance criterion ("allows the skip when there is sufficient buffer") is met only for the trailing-job shape. For mid-chain assigned jobs the admin is still blocked, and the diagnostic they receive regressed from a specific, actionable message to a generic one that misattributes the cause to bad data.

**Fix:** Two options. Minimal — carry the clamp forward so downstream survivors are never pulled behind a job that has already been held, which keeps the chain monotonic by construction and lets the backstop revert to a pure malformed-input guard:

```typescript
let previousNewMaxMs = -Infinity;
// ...
const clampedMs =
  node.returnTimeMs != null
    ? Math.max(collapsedMs, node.returnTimeMs)
    : collapsedMs;

// A held job pushes its successors out too — the chain must stay
// strictly increasing, and collapsing is never mandatory.
const newMaxMs = Math.max(clampedMs, previousNewMaxMs + MIN_GAP_MS);

previousNewMaxMs = newMaxMs;
```

Alternatively, if pushing successors out is not acceptable, keep the refusal but restore a message that names the offending job and points at the workaround, rather than reporting it as broken ordering.

---

### 2. The clamp has no upper bound, so a skip can push a downstream hard stop *later* than it was

**[File: apps/creative-portal/api/utils/jobs/planJobRemovalTiming.ts]**

> **In plain terms:** Skipping a job is meant to only ever free up time. On an order where a job's user due date was previously moved past its own cut-off, skipping an earlier job now moves that job's cut-off *outwards* — giving the order more time than it had, silently, with no warning that the order deadline may now be exceeded. Nobody asked for that, and the admin doing the skip has no way to see it happened.

**Function/Class:** `planJobRemovalTiming`

**Severity:** medium

**Confidence:** medium-high

**Steps to reproduce:**

1. Log in as an Admin and open an order with Editing → QA → Return, where **Return is assigned**.
2. In the order side panel, use the **user due date** picker on an upstream job in Shift mode to push its date out. The shift is applied to every following job's user due date by the same amount, and the only cut-off check made is against the *picked* job's own cut-off — so Return's user due date can end up past Return's cut-off.
3. Skip the **QA** job.
4. **Expected:** Return's cut-off either stays where it is or moves earlier. A skip never grants more time than the order already had.
5. **Actual:** Return's cut-off is rewritten *later* than it was, to match its user due date.

**Problem:** `Math.max(collapsedMs, node.returnTimeMs)` is unbounded above. It is only ever intended to stop the collapse at the committed deadline, but when `returnTime` already exceeds `maxReturnTime` it *raises* the hard stop instead. The write guard `newMaxMs !== originalMaxMs` does not catch this — the value differs, so the write is issued.

**Evidence:** `planJobRemovalTiming.ts:114-128`:

```typescript
const newMaxMs =
  node.returnTimeMs != null
    ? Math.max(collapsedMs, node.returnTimeMs)
    : collapsedMs;
// ...
if (cumulativeRemovedMs > 0 && newMaxMs !== originalMaxMs) {
  downstreamUpdates.push({
    id: node.id,
    maxReturnTime: new Date(newMaxMs).toISOString()
  });
}
```

Executed against the PR's source with Editing `+1h` → QA `+3h` (removed) → Return `maxReturnTime: +3h20m`, `returnTime: +4h20m`, `status: "Assigned"`:

```
{"ok":true,"downstreamUpdates":[{"id":3,"maxReturnTime":"2026-05-13T16:20:00.000Z"}]}
```

`16:20Z` is `NOW + 260m` — **60 minutes later** than the original `+200m` hard stop. Pre-PR, this same input hit the old guard and refused the skip, so the outward write is new behaviour introduced here.

Reachability of the `returnTime > maxReturnTime` precondition: the sidebar user-due-date picker validates only the *selected* job against its own hard stop (`OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturnBase.ts:133-145`), while `updateJobReturnTimes` shifts every following job's `returnTime` by the same delta with no per-job cut-off check (`OrderManagment/partials/OrderJobs/utils.ts:262-274`). The remaining downstream guards are a sequence check on `returnTime`s (preserved by a uniform shift) and an order-deadline *warning* modal that proceeds on confirm. That is a pre-existing gap in a file this PR does not touch, but it is what makes this input state producible.

**Impact:** A skip can extend an order's schedule outwards without any admin intent or visibility — the opposite of the operation's contract, and a silent loosening of deadline governance. It also bypasses the order-deadline warning the picker path shows for the equivalent manual change.

**Fix:** Bound the clamp by the job's original hard stop, so the value can only ever stay put or move earlier:

```typescript
// Never above the original: a skip may hold a hard stop, never extend it.
const newMaxMs =
  node.returnTimeMs != null
    ? Math.min(
        originalMaxMs,
        Math.max(collapsedMs, node.returnTimeMs)
      )
    : collapsedMs;
```

---

### 3. The only refusal-path test was deleted and nothing replaced it

**[File: apps/creative-portal/api/utils/jobs/planJobRemovalTiming.test.ts]**

> **In plain terms:** The change removed the one automated check that confirmed a skip gets blocked when it should be, and did not add a new one. So there is now nothing guarding the single remaining situation where the system refuses a skip — including the wording of the message the admin sees. If a future change silently stops blocking, or starts blocking the wrong things, no test will notice.

**Function/Class:** `describe("planJobRemovalTiming")`

**Severity:** medium

**Confidence:** high

**How it spots it:** Code health — not user-reproducible. `grep -c "^  it("` on the PR's test file returns **12**; `grep "ok).toBe(false)"` returns **nothing**. On `develop` the same grep matches at line 340 — the test this PR converted.

**Problem:** The PR simultaneously (a) promotes the strictly-increasing backstop to *"the one path that still refuses the skip"* which *"now guards a real case, not just malformed input"*, and (b) removes every assertion that any refusal path works. The refusal branch at `planJobRemovalTiming.ts:143-148` and its user-facing `reason` string have zero coverage. Both callers surface that string directly to admins (`OrderManagment/hooks.tsx:334`, `bulkOrderActionsContext/provider.tsx:641-648`), so it is not internal-only.

Per CLAUDE.md the change to the backstop's behaviour is new code and requires a test. The two tests that *were* added are good — the PR verified they fail against the pre-fix source, so they are not vacuous — but they cover only the positive path.

**Impact:** The behaviour the PR calls its most important remaining guard is untested, as is the message admins depend on. Both fixes proposed in Issue 1 and Issue 2 change this branch, and there is nothing to regression-test them against.

**Fix:** Add a test for the clamp-induced ordering break (this doubles as the reproduction for Issue 1), and one for the bounded clamp once Issue 2 is fixed:

```typescript
it("refuses when clamping one job drags the chain out of order", () => {
  const jobs = [
    buildJob({ id: 1, jobType: JobType.SERVICE, maxReturnTime: after(60) }),
    buildJob({ id: 2, jobType: JobType.REVIEW, maxReturnTime: after(600) }),
    buildJob({
      id: 3,
      jobType: JobType.QA,
      status: "Assigned",
      maxReturnTime: after(660),
      returnTime: after(660)
    }),
    buildJob({ id: 4, jobType: JobType.RETURN, maxReturnTime: after(680) })
  ];

  const result = planJobRemovalTiming({
    jobs,
    removedJobIds: [2],
    now: NOW
  });

  expect(result.ok).toBe(false);
  if (result.ok) return;
  expect(result.reason).toMatch(/ordering/i);
});
```

---

### 4. Both callers still document the refusal reason this PR removed

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/hooks.tsx]**

> **In plain terms:** Notes left in two other files still describe the old blocking rule as if it were still in place. Anyone reading them to understand when a skip gets refused will be given the wrong explanation.

**Function/Class:** `removeJobAndUpdateReturnTimes`, `handleDeleteJobs`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health — not user-reproducible. Three comments describe a branch that no longer exists:

- `OrderManagment/hooks.tsx:321-323` — *"so a conflict (an assigned downstream job whose deadline would fall outside its new hard stop) aborts cleanly with nothing deleted"*
- `OrderManagment/hooks.tsx:332-333` — *"Surface the planner's specific reason (committed-deadline conflict vs. broken job ordering)"* — the committed-deadline reason is gone; ordering is the only one left
- `contexts/bulkOrderActionsContext/provider.tsx:609-611` — *"An order whose reflow is blocked (an assigned downstream job's committed deadline would be exceeded) is left fully untouched"*

**Problem:** The PR carefully updated the planner's own comments (including correcting the "provably preserved" claim, which is the right call) but did not sweep the two call sites that restate the removed rule.

**Impact:** Misleading documentation at exactly the points where a future reader looks to understand skip refusals. No runtime effect.

**Fix:** Update all three to describe the ordering backstop as the sole refusal path. If Issue 1 is addressed by carrying the clamp forward, these comments should describe whatever refusal remains at that point.

---

## Open Questions

- Order 53159's four `maxReturnTime` / `returnTime` values — the PR flags this itself. If the Return job on 53159 had a downstream job, or if its buffer shape differs from 21369's, Issue 1 may mean the originally-reported order is still blocked after this fix. Worth confirming before closing the ticket.
- `bulkOrderActionsContext/provider.tsx:641-648` joins distinct `blockedReasons` into one toast. With the committed-deadline reason removed, every refusal now produces the identical ordering string, so the `Set` collapses to a single message across all blocked orders and the admin cannot tell which orders were affected. Is that acceptable, or should the message name the orders?
- The PR checklist leaves `/security` unticked and `yarn bump-packages` unrun. `/security` is arguably unnecessary for a pure planner with no new request surface (as the PR argues), but CLAUDE.md mandates it before opening a PR — is the team treating it as waivable for changes with no new input surface?

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⚠️ Partial | Skipped — user opted out of the full suite. The changed module *was* run directly against the PR's source in a clean tree: **12/12 pass** (`api/utils/jobs/planJobRemovalTiming.test.ts`, 22ms). |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped — user opted out. |
| `npx turbo run lint` | ⏭️ Skipped | Skipped — user opted out. |
| `npx turbo run build` | ⏭️ Skipped | Skipped — user opted out. |

The full suite was **not** run — re-run `npx turbo run test typecheck lint build --filter=@proofed/creative-portal` before merging. The diff introduces no exported-signature changes (`PlanJobRemovalResult` and `DownstreamMaxReturnUpdate` are untouched), so type/lint risk is low, but this is not a substitute for the gate.

---

## Tests

- ✅ Two new tests encode the fix, and the PR verified both fail against the pre-fix source — not vacuous.
- ✅ The converted test's assertions are specific and behavioural (`downstreamUpdates` equals an exact array), not snapshot-style or "renders without throwing".
- ✅ The zero-slack case from order 21369 is captured, including the "no write when fully clamped" assertion.
- ✅ The module's 12 tests pass against the PR source.
- ❌ No test covers any refusal path — the only `ok: false` assertion in the module was removed (Issue 3).
- ❌ No test covers the clamp-induced ordering break, which is the failure mode Issue 1 identifies as reachable.
- ❌ No test covers `returnTime > maxReturnTime` — the input that produces the outward write in Issue 2.
- ⚠️ Neither caller (`OrderManagment/hooks.tsx`, `bulkOrderActionsContext/provider.tsx`) has a test exercising the refusal toast, so the reason string is unverified end-to-end.

### Suggested manual QA script

1. **Verifies the fix (Issue-free path).** On a test order with Editing → QA → Return where **Return is assigned** and its tray buffer shows `0h 0m`: skip QA. Expect the skip to succeed with no error toast, and Return's Job Due Date to be unchanged.
2. **Verifies Issue 1.** On an order with Editing → Review → QA → Return where **QA is assigned** with little or no buffer, and Return follows QA closely: skip the Review job (use the orders-table **More → Skip** on the current job so QA is not auto-paired). Expect — per the ticket — the skip to succeed. Report the exact toast text if it is refused.
3. **Verifies Issue 2.** On an order with Editing → QA → Return where **Return is assigned**: first use the side panel's user due date picker on Editing in Shift mode to push dates out until Return's user due date sits past its Job Due Date. Note Return's Job Due Date. Then skip QA and re-check it. Expect it to be unchanged or earlier; report it if it has moved **later**.
4. **Ticket acceptance check not covered by unit tests.** Confirm against order 53159's actual four `maxReturnTime` / `returnTime` values that its shape matches case 1 above and not case 2 — this is the assumption the PR asks to have checked.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ Fix is correct for the reported shape; incomplete for mid-chain assigned jobs, and unbounded above |
| Regression risk | ✅ Low — nothing previously allowed becomes blocked; refusals are planned before any write, so no partial state |
| Tests | ⚠️ Good positive coverage, zero negative coverage |
| Accessibility | n/a — no UI change |
| Error handling | ⚠️ The one remaining refusal message misdescribes its most likely cause |
| Security | ✅ Pure planner, no new request surface, no new inputs (`/security` still unticked on the PR checklist) |
| Code quality | ✅ Clear, well-commented, correctly diagnosed; comments in the two callers are now stale |
| Validation suite | ⚠️ Not run — user opted out; changed module verified directly (12/12) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions** — the diagnosis is correct, the change is well-argued, and it fixes the reported case. Nothing here risks writing bad data: every failure mode ends in a refusal before any write. But two of the four findings should be settled before merge.

1. **Bound the clamp** (Issue 2) — a one-line `Math.min(originalMaxMs, …)`. A skip must never extend a hard stop. Lowest-cost, highest-certainty fix in this review.
2. **Decide on mid-chain assigned jobs** (Issue 1) — either carry the clamp forward so the chain stays monotonic, or keep the refusal and restore a message that names the job and points at the workaround. The current generic ordering message is a diagnostic regression for that shape.
3. **Add a refusal-path test** (Issue 3) — the module currently has none, and both fixes above touch that branch.
4. **Sweep the three stale caller comments** (Issue 4) — `OrderManagment/hooks.tsx:321-323`, `:332-333`, `bulkOrderActionsContext/provider.tsx:609-611`.
5. **Run the validation gate** before merging — `npx turbo run test typecheck lint build --filter=@proofed/creative-portal`. It was skipped for this review.
6. **Answer the author's own open question to the PO** — the PR explicitly asks whether "a skip should always be allowed when the order deadline still fits" is intended, since the block was deliberate. Issue 1 makes that question sharper: as written, the answer is "yes, but only for the last job in the chain."
