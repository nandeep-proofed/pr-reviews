# PR Review: fix/PP-2020: stop refusing a QA skip that would only tighten the schedule

**PR:** https://github.com/Proofed/B2BWebserver/pull/2431

**Jira:** https://proofed.atlassian.net/browse/PP-2020

**Status:** Code Review — **all four findings resolved in `39f7c25de`**

**Author:** nandeep-proofed · **Branch:** `fix/PP-2020-skip-blocked-by-committed-deadline` → `develop`

**Reviewed at:** `ec5b1f4` (2 files, +66 / −23) · **Re-verified at:** `39f7c25de` (4 files, +190 / −8 on top)

---

## Resolution summary

| # | Finding | Severity | Status |
| --- | --- | --- | --- |
| 1 | Hold only rescues the last job in the chain; a mid-chain assigned job still refuses, with a worse message | medium | ✅ Fixed in `39f7c25de` |
| 2 | The hold has no upper bound, so a skip can push a hard stop *later* | medium | ✅ Fixed in `39f7c25de` |
| 3 | The only refusal-path test was deleted and nothing replaced it | medium | ✅ Fixed in `39f7c25de` |
| 4 | Both callers still document the refusal reason the PR removed | low | ✅ Fixed in `39f7c25de` |

Findings 1 and 2 were fixed by one mechanism rather than two patches. See **How the fixes landed** below. The original findings are kept in full as the record of why the change looks the way it does.

Three open questions remain, none of them code changes. They are listed at the end.

---

## What this means for users (non-technical summary)

1. **The reported problem is genuinely fixed.** When the last job in an order is already booked out to someone and has no spare time, an admin can now skip an earlier job instead of being blocked. Verified by running the changed code.
2. ~~The same block still happens when the booked-out job is in the middle of the order.~~ **Fixed.** A booked-out job in the middle of the order now holds where it is and everything after it holds too, so the skip goes through instead of failing with a message about the order's dates being out of order.
3. ~~In one situation a skip can now push a job's cut-off time *later* than it was.~~ **Fixed.** A skip can now only ever leave a cut-off where it is or move it earlier, never later.
4. ~~No automated test covers any situation where a skip is refused.~~ **Fixed.** The one remaining refusal case is now covered, along with the message an admin sees.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| Skipping a Review/QA job "shifts subsequent jobs' deadlines proportionately" | `planJobRemovalTiming` still collapses surviving downstream `maxReturnTime` values by the removed span; unchanged behaviour for unassigned downstream jobs | ✅ Addressed |
| "…and allows the skip when there is sufficient buffer" | The `returnTimeExceedsMax`-style refusal is replaced by a hold at the committed `returnTime`, and `39f7c25de` carries the reduced collapse forward so the hold works anywhere in the chain, not just at the tail | ✅ Addressed |
| Root cause identified (intermittent / "similar orders skipped QA fine") | Correctly diagnosed: the check only fires when `returnTime` is set, which requires assignment. The PR's inequality and the "assigned vs unassigned" explanation match the code | ✅ Addressed |
| Reproduce against reported order 53159 | Reproduced on B2B test order 21369 instead; the PR itself flags 53159 as unconfirmed | ⚠️ Partial — author already called this out for the PO |

**Scope:** tight and single-purpose. The diff touches the planner, its test file, and two comment-only caller edits. Title and branch follow `fix/PP-XXXX: <subject>`.

---

## Architecture Analysis

`planJobRemovalTiming` is a pure planner: given the order's pre-delete jobs and the ids being removed, it returns the net `maxReturnTime` changes for surviving downstream jobs. Two callers consume it, both planning **before** any write, so a refusal leaves the order untouched:

- `components/organisms/sidebars/contents/OrderManagment/hooks.tsx:323` — single-order skip from the sidebar
- `contexts/bulkOrderActionsContext/provider.tsx:628` — bulk skip, per-order, blocked orders reported and the rest proceed

The change swaps a hard refusal for a hold at the assigned job's committed `returnTime`, and suppresses the write when the hold is total. The reasoning in the PR description is sound and the diagnosis is correct.

**The gap as originally reviewed:** the hold was applied **per job** while the ordering invariant it breaks is **chain-wide**. The strictly-increasing backstop caught the resulting inconsistency and refused the whole skip, so nothing corrupt was ever written, but the fix was complete only where the held job was last in the chain.

**How `39f7c25de` closes it:** the running pull-forward is cut back to what each surviving job actually absorbed (`pullForwardMs = originalMaxMs - newMaxMs`). A held job therefore pins its successors instead of being overtaken by them, ordering is preserved by construction again, and the backstop reverts to guarding malformed input. The alternative sketched in the original Issue 1 (pushing successors outward) was **not** taken: it would have extended hard stops, which is exactly what Issue 2 says must never happen, and it referenced a `MIN_GAP_MS` constant that does not exist in the repo.

**Verification method:** the PR's source files were checked out into a clean working tree and the planner was exercised directly with `vitest`, both at `ec5b1f4` (to produce the evidence quoted in Issues 1 and 2) and at `39f7c25de` (to confirm the fixes).

---

## Issues Found

### 1. The hold only rescues the last job in the chain — a mid-chain assigned job still refuses the skip, with a worse message

**[File: apps/creative-portal/api/utils/jobs/planJobRemovalTiming.ts]**

**Severity:** medium · **Confidence:** high · **Status: ✅ Fixed in `39f7c25de`**

> **In plain terms:** The fix worked when the booked-out job was the final job on the order, which is the case that was reported. If a booked-out job sat in the middle instead, the admin was still blocked, and the message said the order's dates were in the wrong order. That reads like a data fault, tells them nothing they can act on, and no longer points at the workaround the ticket records ("manually shift the deadlines forward, then skip").

**Problem:** The hold was applied independently per job, but the ordering invariant it violates is chain-wide. Once an upstream survivor was held at its `returnTime`, the next survivor was still collapsed by the full `cumulativeRemovedMs` and landed **before** it, so the strictly-increasing backstop rejected the entire operation.

**Evidence (at `ec5b1f4`):** executed against the PR's source with a chain of Editing `+1h` → Review `+10h` (removed) → QA `+11h` (`status: "Assigned"`, `returnTime: +11h`) → Return `+11h20m`:

```
{"ok":false,"reason":"Skipping would break job due-date ordering."}
```

QA held at `+11h`; Return collapsed to `680m − 540m = +2h20m`, earlier than QA, hence the refusal. The condition is `removedSpan > (nextJob.max − heldJob.max)`, i.e. the removed job's window exceeds the gap to the following job. With Review windows measured in hours and QA→Return gaps in tens of minutes, that inequality is the normal case rather than an exotic one, so **whenever a held job had a surviving job after it, this would essentially always trip.**

Reachability of a ≥2-survivor chain: the bulk **Skip current job** action passes `currentJobIds` with no Review+QA pairing (`components/molecules/tables/TableWithFilters/utils/getBulkActionsConfig.tsx:196-208`, gated by `isAbleToSkipCurrentJob` = every current job has `allowDeletion`, `contexts/bulkOrderActionsContext/provider.tsx:453-457`). Skipping a current Review there leaves QA **and** Return downstream. The sidebar's Review+QA pairing (`OrderManagment/hooks.tsx:309-316`) avoids this for the specific Review case, which is why the reported order 21369 was fixed.

**Fix applied:** the pull-forward is cut back to what each survivor absorbed, so a held job pins everything after it:

```typescript
const collapsedMs = originalMaxMs - pullForwardMs;

const newMaxMs =
  node.returnTimeMs != null
    ? Math.min(originalMaxMs, Math.max(collapsedMs, node.returnTimeMs))
    : collapsedMs;

// Whatever this job absorbed is the most any later job may absorb.
pullForwardMs = originalMaxMs - newMaxMs;
```

Re-run at `39f7c25de`, the same chain returns `ok: true` with no updates: QA holds at `+11h` and Return keeps its 20m gap behind it. Covered by `does not pull a later job in front of one held at its deadline` and, for the partial case, `carries the reduced collapse past a partially held job`.

---

### 2. The hold has no upper bound, so a skip can push a downstream hard stop *later*

**[File: apps/creative-portal/api/utils/jobs/planJobRemovalTiming.ts]**

**Severity:** medium · **Confidence:** medium-high · **Status: ✅ Fixed in `39f7c25de`**

> **In plain terms:** Skipping a job is meant to only ever free up time. On an order where a job's user due date was previously moved past its own cut-off, skipping an earlier job moved that job's cut-off *outwards*, giving the order more time than it had, silently, with no warning that the order deadline may now be exceeded.

**Problem:** `Math.max(collapsedMs, node.returnTimeMs)` was unbounded above. It is only ever intended to stop the collapse at the committed deadline, but when `returnTime` already exceeded `maxReturnTime` it *raised* the hard stop instead. The write guard `newMaxMs !== originalMaxMs` did not catch this: the value differs, so the write was issued.

**Evidence (at `ec5b1f4`):** executed with Editing `+1h` → QA `+3h` (removed) → Return `maxReturnTime: +3h20m`, `returnTime: +4h20m`, `status: "Assigned"`:

```
{"ok":true,"downstreamUpdates":[{"id":3,"maxReturnTime":"2026-05-13T16:20:00.000Z"}]}
```

`16:20Z` is `NOW + 260m`, **60 minutes later** than the original `+200m` hard stop. Pre-PR, this same input hit the old guard and refused the skip, so the outward write was new behaviour introduced by `ec5b1f4`.

Reachability of the `returnTime > maxReturnTime` precondition: the sidebar user-due-date picker validates only the *selected* job against its own hard stop (`OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturnBase.ts:130-145`), while `updateJobReturnTimes` shifts every following job's `returnTime` by the same delta with no per-job cut-off check (`OrderManagment/partials/OrderJobs/utils.ts:262-275`). The remaining downstream guards are a sequence check on `returnTime`s (preserved by a uniform shift) and an order-deadline *warning* modal that proceeds on confirm.

**Fix applied:** `Math.min(originalMaxMs, ...)` bounds the hold, so the value can only stay put or move earlier. Re-run at `39f7c25de`, the same input returns `ok: true` with **no** update for the Return job. Covered by `never pushes a hard stop later than it already was`.

**Note:** the precondition itself is a pre-existing gap in a file this PR does not touch. The planner now refuses to make it worse, but the picker still permits producing that state. Flagged in the PR description as deserving its own ticket.

---

### 3. The only refusal-path test was deleted and nothing replaced it

**[File: apps/creative-portal/api/utils/jobs/planJobRemovalTiming.test.ts]**

**Severity:** medium · **Confidence:** high · **Status: ✅ Fixed in `39f7c25de`**

> **In plain terms:** The change removed the one automated check that confirmed a skip gets blocked when it should be, and did not add a new one. Nothing guarded the single remaining situation where the system refuses a skip, including the wording of the message the admin sees.

**Problem:** `ec5b1f4` simultaneously (a) promoted the strictly-increasing backstop to "the one path that still refuses the skip" and (b) removed every assertion that any refusal path works. `grep -c "^  it("` returned **12**; `grep "ok).toBe(false)"` returned nothing. On `develop` the same grep matched at line 340, the test the PR converted. Both callers surface the `reason` string directly to admins, so it is not internal-only.

**Fix applied:** `refuses the skip when the input hard stops are out of order` asserts both `ok: false` and the exact reason string, using two surviving jobs that share a `maxReturnTime`. The module is now at **16** tests. This test passes against both the pre-fix and post-fix source by design: it guards a path that already existed and must keep existing, rather than encoding new behaviour.

---

### 4. Both callers still document the refusal reason this PR removed

**[Files: OrderManagment/hooks.tsx, bulkOrderActionsContext/provider.tsx]**

**Severity:** low · **Confidence:** high · **Status: ✅ Fixed in `39f7c25de`**

Three comments described a branch that no longer exists:

- `OrderManagment/hooks.tsx:320-322` — *"so a conflict (an assigned downstream job whose deadline would fall outside its new hard stop) aborts cleanly with nothing deleted"*
- `OrderManagment/hooks.tsx:331-332` — *"Surface the planner's specific reason (committed-deadline conflict vs. broken job ordering)"*
- `contexts/bulkOrderActionsContext/provider.tsx:609-611` — *"An order whose reflow is blocked (an assigned downstream job's committed deadline would be exceeded) is left fully untouched"*

**Fix applied:** all three rewritten to describe out-of-order input hard stops as the sole refusal path, with the sidebar comment noting that an assigned job inside the collapse now holds where it is.

---

## How the fixes landed

`39f7c25de` — *fix/PP-2020: carry the reduced collapse past a job held at its deadline*

| Change | Closes |
| --- | --- |
| `pullForwardMs` cut back to what each survivor absorbed | Issue 1 |
| `Math.min(originalMaxMs, ...)` bound on the hold | Issue 2 |
| Write guard simplified to `newMaxMs !== originalMaxMs` (the `pullForwardMs > 0` half became redundant, and the bound makes it correct for upstream jobs with a stale `returnTime`) | Issue 2 |
| Backstop comment rewritten to explain *why* ordering is preserved rather than assert it | Issue 1 |
| Four new tests: held mid-chain, partially held, never-extend, malformed-input refusal | Issues 1, 2, 3 |
| Three caller comments rewritten | Issue 4 |

The three behavioural additions were run against the pre-fix source and all fail there, so they are not vacuous:

```
× does not pull a later job in front of one held at its deadline
× carries the reduced collapse past a partially held job
× never pushes a hard stop later than it already was
   Tests  3 failed | 13 passed (16)
```

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `planJobRemovalTiming` module | ✅ Pass | 16/16 |
| Impact radius | ✅ Pass | 259 tests across 28 files (removal/insertion planners, OrderManagment side panel, bulk order actions) |
| `npx turbo run typecheck --filter=@proofed/creative-portal` | ✅ Pass | clean |
| `npx turbo run lint --filter=@proofed/creative-portal` | ✅ Pass | clean, `--max-warnings 0` |
| `npx turbo run build --filter=@proofed/creative-portal` | ✅ Pass | clean |
| `npx turbo run test` (full monorepo suite) | ⏭️ Not run | Scoped to the impact radius per CLAUDE.md's pre-commit guidance. Run it before release, not per-change. |

---

## Tests

- ✅ Five tests encode the fix, all verified failing against the pre-fix source.
- ✅ Assertions are specific and behavioural (`downstreamUpdates` equals an exact array), not snapshot-style.
- ✅ The zero-slack case from order 21369 is captured, including the "no write when fully held" assertion.
- ✅ The refusal path is covered, message string included.
- ✅ Both the fully-held and partially-held mid-chain shapes are covered, so the carry-forward is pinned in both directions.
- ⚠️ Neither caller has a test exercising the refusal toast, so the reason string is unverified end-to-end. Low value now that the refusal is malformed-input only.

### Suggested manual QA script

1. **Verifies the reported fix.** On a test order with Editing → QA → Return where **Return is assigned** and its tray buffer shows `0h 0m`: skip QA. Expect the skip to succeed with no error toast, and Return's Job Due Date unchanged.
2. **Verifies the mid-chain fix.** On an order with Editing → Review → QA → Return where **QA is assigned** with little or no buffer, and Return follows QA closely: skip the Review job (use the orders-table **More → Skip** on the current job so QA is not auto-paired). Expect the skip to succeed, with QA and Return holding their dates.
3. **Verifies the never-extend fix.** On an order with Editing → QA → Return where **Return is assigned**: first use the side panel's user due date picker on Editing in Shift mode to push dates out until Return's user due date sits past its Job Due Date. Note Return's Job Due Date. Then skip QA and re-check it. Expect it unchanged. Report it if it has moved **later**.
4. **Ticket acceptance check not covered by unit tests.** Confirm against order 53159's actual four `maxReturnTime` / `returnTime` values that its shape matches case 1 or case 2 above.

---

## Open Questions

None of these are code changes, and none block merge on correctness grounds.

- **Order 53159's four `maxReturnTime` / `returnTime` values.** The PR flags this itself. With Issue 1 fixed, both the trailing and mid-chain shapes are handled, so the risk that 53159 differs from 21369 in a way that matters is much lower than at first review. Still worth confirming before closing the ticket.
- **The bulk toast.** `bulkOrderActionsContext/provider.tsx` joins distinct `blockedReasons` into one message. With the committed-deadline reason removed, every refusal now produces the identical ordering string, so the `Set` collapses to a single line across all blocked orders. The count is reported but not which orders. Acceptable, or should it name them?
- **The user-due-date picker gap.** `returnTime` can be shifted past a downstream job's own `maxReturnTime` (see Issue 2's reachability note). The planner no longer compounds it; the picker still permits it. Separate ticket.
- **Process.** The PR checklist leaves `/security` and `yarn bump-packages` unticked. `/security` is arguably unnecessary for a pure planner with no new request surface, but CLAUDE.md mandates it before opening a PR. Is the team treating it as waivable for changes with no new input surface?

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ✅ Fixed for every shape identified: trailing, mid-chain, partially held, and the never-extend case |
| Regression risk | ✅ Low. Nothing previously allowed becomes blocked; refusals are planned before any write, so no partial state |
| Tests | ✅ 16 in the module, positive and negative paths, non-vacuous |
| Accessibility | n/a — no UI change |
| Error handling | ✅ The one remaining refusal message now matches the only condition that produces it |
| Security | ✅ Pure planner, no new request surface, no new inputs (`/security` still unticked on the PR checklist) |
| Code quality | ✅ Clear, well-commented, correctly diagnosed; caller comments now consistent with the planner |
| Validation suite | ✅ Test / typecheck / lint / build all clean on `@proofed/creative-portal` |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve.** All four review findings are fixed in `39f7c25de` and re-verified by executing the planner, not by reading it. The diagnosis was right from the start; the follow-up commit generalises the remedy from "the tail of the chain" to "anywhere in the chain" and adds the one guarantee the first cut was missing, that a skip may hold a hard stop but never push it out.

Remaining before merge is judgement, not code:

1. **Answer the author's question to the PO** — the PR asks whether "a skip should always be allowed when the order deadline still fits" is intended, since the block was deliberate. With Issue 1 fixed, the answer is now uniformly "yes" rather than "yes, but only for the last job in the chain."
2. **Confirm order 53159** against its actual timing values, to retire the last assumption in the PR.
3. **Decide on the bulk toast wording** and whether the user-due-date picker gap gets its own ticket.
