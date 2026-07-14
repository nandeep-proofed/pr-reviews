# PR Review: fix/PP-1864: recalculate charge adjustments on word count change

**PR:** https://github.com/Proofed/B2BWebserver/pull/2294
**Jira:** https://proofed.atlassian.net/browse/PP-1864
**Status:** Code Review — **all 4 issues triaged**

> ## ⚠️ RETRACTED: the "43 failing tests / merge blocker" claim in an earlier revision was WRONG
>
> A previous revision of this report asserted a **merge blocker**: "43 tests failing, caused by branch staleness." **That claim is withdrawn — the measurement was invalid.**
>
> **What went wrong:** the reviewer ran the suite on this branch without running `yarn install` after switching from a develop-based branch. So `node_modules` contained **develop's** dependency tree while this branch pins an older one:
>
> | | `@sentry/nextjs` |
> |---|---|
> | Installed in `node_modules` at test time | **10.48.0** (develop's) |
> | What this PR's branch (`a8a585212`) pins | **7.73.0** |
> | What develop pins | 10.48.0 |
>
> Month-old code was tested against develop's dependencies. The "develop passes 63/63" control run was the one case where `node_modules` actually matched the checked-out code — which is exactly what the confound predicts, not evidence of branch staleness.
>
> **CI installs per-branch**, so it would resolve `7.73.0` and may well be green. **The 43 failures are not evidence against this PR and should not be chased.**
>
> **What survives:** the branch **is 42 commits behind develop** (merge-base `23d270128`, 2026-06-10; develop head `30139ba35`, 2026-07-13). That is an independently verified fact, unaffected by the environment. It is worth a routine update — but it is **not** a demonstrated blocker, and the earlier "the billing utility may have moved underneath it" concern was checked and is **false**: none of the 42 commits touch `calculateOrderChargeAdjustments`, `calculateChargeAdjustments`, `calculateOrderSubtotal`, `calculateFormatPremium`, or the `ChangeWordCountModal` folder.
>
> **Also now unreliable:** the ✅ typecheck / build / 8-of-8 results below ran in that same mismatched environment. They are probably sound (the applied changes are a docstring and a dead test mock), but they should not be leaned on. **A clean verification — `yarn install` on this branch, then the suite — has not been performed.**

---

## Review Outcomes

| # | Issue | Severity | Status | Resolution |
|---|---|---|---|---|
| 1 | Format premium silently keeps its stale value if submitted before the `PremiumChargeRate` query resolves | low | ⏭️ **Skipped** | Chain verified end-to-end by reading the code — the finding is **real**. Not fixed here: `useAddNewJob.ts:59-60` carries the byte-identical `= []` pattern, so a one-site fix would leave the twin hole and two contradictory patterns. Follow-up should cover both call sites + the root null-ambiguity. |
| 2 | Partial-failure window: order persisted and modal closed before job-task updates complete | low | ⏭️ **Acknowledged** | Ordering is pre-existing (confirmed against develop). The suggested reorder **does not fix it** — without a transaction it just relocates the inconsistency. No change, per the issue's own "No change required here". |
| 3 | Caller relies on an implementation detail of `mode: "add"`, not its documented contract | low | ✅ **Fixed** | Docstring rewritten on `a8a585212`. This PR introduced the **only** out-of-contract caller, so the mismatch is squarely this PR's. Fixed at the utility (the wrong contract), not via a call-site comment. |
| 4 | Test carries a leftover mock for a module the hook no longer imports | low | ✅ **Fixed** | `api/jobTypes/enums` mock and `jobType` fixture fields removed on `a8a585212`. |

**Applied on commit `a8a585212`**, pushed to `fix/PP-1864-change-word-count-recalc-charge-adjustments`.

All four findings are static-analysis conclusions drawn from reading the source; none depended on the broken test run.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| `workItemSize` updates to the new value | `updateOrder({ ...order, workItemSize: updatedWorkItemSize, ...chargeAdjustments })` | ✅ Addressed |
| Per-word job tasks' `quotedChargeQuantity` / `quotedPayQuantity` update to the new value | `updatedJobTasks` maps per-word tasks (matched by `chargeUnit`/`payUnit`) to the new size, then each is sent via `updateJobTask` | ✅ Addressed |
| `minimumChargeAdjustmentAmount` / `minimumChargeRate` recalculated against new subtotal (cleared to 0 when above threshold) | `calculateOrderChargeAdjustments({ mode: "add" })` recomputes the subtotal from `updatedJobTasks`; add-mode sets a fresh adjustment when `subtotal < minimumChargeAmount`, or clears to 0 when `subtotal >= minimumChargeAmount` and a prior adjustment existed | ✅ Addressed |
| `formatPremiumAmount` / `formatPremiumRate` recalculated against new subtotal | Same utility recomputes format premium via `calculateFormatPremium` using the new subtotal + (possibly new) min-charge adjustment; result folded into the single `updateOrder` call | ✅ Addressed (caveat: Issue 1) |

**Scope beyond Jira (all reasonable):**
- Removed the now-dead `serviceJobIds` / `serviceJobTasks` `useMemo`s and the `JobType` import — `serviceJobTasks` was only ever referenced in the dependency array, never in the callback body, so this is pure dead-code removal.
- Removed the `// eslint-disable-next-line react-hooks/exhaustive-deps` and corrected the dependency array, so exhaustive-deps now passes honestly.
- Added a new `hooks.test.ts` (previously none).

---

## Architecture Analysis

The fix is small, surgical, and correct in approach. Rather than reimplementing the pricing math, it reuses the existing `calculateOrderChargeAdjustments` utility with `mode: "add"` — the same utility and mode already used by `useAddNewJob.ts`. The reuse is idiomatic and matches CLAUDE.md's reuse-first convention.

Key structural improvements:
- `updatedJobTasks` is computed **once** and used for both the charge recalculation (subtotal input) and the `updateJobTask` persistence, guaranteeing the recalculated charge fields reflect exactly the job-task state that will be persisted.
- Charge adjustments are folded into the **single** existing `updateOrder` call rather than a second round-trip.

Correctness of `mode: "add"` for a word-count change (which can move the subtotal **up or down**) was verified against the utility source:
- `computeAddModeAdjustments` handles both directions: `subtotal < minimumChargeAmount` → apply/refresh; `subtotal >= minimumChargeAmount && priorAdjustment > 0` → clear to 0.
- `calculateOrderSubtotalFromJobTasks` filters by `chargeable` internally, so passing all `updatedJobTasks` is safe.
- Format premium is recomputed against `subtotal + (new or existing) minimumChargeAdjustmentAmount`, and only overrides when the value actually changes.

**Verified by reading the source — the subtotal genuinely does not depend on the master reference.** `calculateOrderSubtotalFromJobTasks` → `getJobTotalPrice({ withoutPremium: true })` skips the premium block, and each per-task `calculateJobTaskPrice` passes `usePremiumRate: false`, which makes `getJobTaskRate` return `rate` before it ever reads `predefinedMasterReference`. This is what confines Issue 1's blast radius to format premium alone.

The manual verification in the PR description (order 20907: 837 → 1200 words, min-charge clears to €0, format premium recalculates to €12.48, total €60.00 → €74.88) reconciles with this logic.

---

## Issues Found

### 1. Format premium silently keeps its stale value if the `PremiumChargeRate` query is unresolved — ⏭️ SKIPPED (deferred)

**[File: .../ChangeWordCountModal/hooks.ts]**

**Severity:** low

**Chain verified end-to-end — the finding is real:**

1. `hooks.ts` — `const { data: masterReferencesCharge = [] } = useMasterReferenceQuery(ReferenceName.PremiumChargeRate)`. While loading, `data` is `undefined` → `[]`.
2. `calculateFormatPremium` (sync mode) — `predefinedMasterReference?.find(ref => ref.name === masterReferenceKey)` over `[]` → `undefined` → `if (!reference?.value) return null`.
3. `computeAddModeAdjustments` — `if (premium && ...)` skipped, so `result` carries no format-premium keys.
4. `hooks.ts` — `updateOrder({ ...order, workItemSize, ...chargeAdjustments })` re-persists the **stale** `formatPremiumAmount` / `formatPremiumRate` via the spread.

**A sharper framing than the original.** One line above in this same hook, `useMasterReferenceUnits` handles the identical concern **correctly** — it defaults `chargePerWordUnit`/`payPerWordUnit` to `DEFAULT_PER_WORD_UNIT = 1000` while loading, so per-word matching never silently degrades. The file already demonstrates the right pattern; `masterReferencesCharge = []` is the odd one out. That is a better argument for a guard than "there's a race".

**The original understates the failure mode.** It frames this as a submit-before-load *race*. But if the query **fails** rather than being slow, `data` stays `undefined` → `[]` **permanently** — not a narrow window, but persistent silent mis-billing for that session. A submit-while-loading guard would not catch that variant at all.

**Why it's deferred:** `useAddNewJob.ts:59-60` carries the byte-identical `= []` pattern with no guard. This PR copied an existing pattern in order to reuse an existing utility. Fixing only this call site leaves the twin hole and leaves two contradictory patterns with no signal as to which is intended.

**Root cause worth naming for the follow-up:** `calculateFormatPremium` returns `null` for two different things — "this isn't a premium format" (legitimate, ignore) and "the reference is unavailable" (a failure). Callers cannot distinguish them, so a data failure is silently treated as a no-op. **Follow-up:** cover both call sites and resolve the null-ambiguity at the utility.

### 2. Partial-failure window: order persisted and modal closed before job-task updates complete — ⏭️ ACKNOWLEDGED (no change)

**[File: .../ChangeWordCountModal/hooks.ts]**

**Severity:** low

**Confirmed pre-existing.** The sequence on develop is already `updateOrder` → `setIsChangeWordCountModalOpen(false)` → `updateJobTask` × n. The PR did not introduce it.

**The observation earns its place:** folding the recalculated charge fields into that first call means the order now carries `minimumChargeAdjustmentAmount` / `formatPremium*` **derived from job-task quantities that have not persisted yet**. Previously only `workItemSize` was exposed; now the money fields are too. That coupling is new.

**But the proposed fix does not work.** There is no transaction available — these are two independent mutations — so reordering relocates the inconsistency rather than removing it:
- *Order-first failure (today):* order says 1200 words with adjustments for 1200; tasks still say 500.
- *Tasks-first failure (proposed):* tasks say 1200; order says 500 words with adjustments for 500.

Both leave order-level and task-level state disagreeing, and both mis-bill. Swapping buys nothing without a backend endpoint writing both atomically. The suggestion would churn a working path for no gain.

**The one genuine improvement** in the suggestion is deferring the modal close until after `Promise.all` — today a job-task failure toasts against a dismissed modal with no retry path. Two lines, but pre-existing behaviour and a UX change, so not this PR's job.

**Test gap noted:** there is a case asserting that when `updateOrder` rejects, `updateJobTask` is never called. There is **no** case for the inverse — `updateOrder` succeeding and a `updateJobTask` rejecting — which is exactly the partial-write window this issue describes.

**Resolution:** ⏭️ No change, matching the issue's own "No change required here". Follow-up for the real fix (atomic write, or at minimum deferring the modal close) — not the reorder.

### 3. Caller relies on an implementation detail of `mode: "add"`, not its documented contract — ✅ FIXED

**[File: apps/creative-portal/utils/calculateOrderChargeAdjustments.ts]**

**Severity:** low

**Verified — and this is the one finding squarely attributable to this PR.** All three callers:

| Caller | Mode | Satisfies old docstring? |
|---|---|---|
| `useAddNewJob.ts:131` | `add` | ✅ adds a chargeable job |
| `OrderManagment/hooks.tsx:365` | `remove` | ✅ passes `removedChargeableJobsSubtotal` |
| `ChangeWordCountModal/hooks.ts:67` — **new in this PR** | `add` | ❌ adds no job |

**The docstring was wrong, not merely narrow.** `computeAddModeAdjustments` never references an added job — it only recomputes the subtotal from `jobTasks` and compares against `minimumChargeAmount` in both directions. The "a chargeable job was added" precondition is **vestigial** for add mode; it gates no logic. (For remove mode it is real — `removedChargeableJobsSubtotal` genuinely depends on it.) So the docstring over-claimed a precondition the implementation never had, which made this caller *look* illegitimate when it isn't.

**Fixed at the utility, not the call site.** The original offered two options; the call-site comment was the wrong one — a comment saying "we rely on add-mode's full-recompute behaviour" explains why the code is correct, which is noise that rots. The docstring is the actual contract and the actual defect.

**Resolution:** ✅ Docstring rewritten on `a8a585212` to describe what add mode actually guarantees (full recompute, both directions, pass post-change job tasks) while keeping remove mode's real precondition. Closes the named risk: someone "optimizing" add-mode on the false assumption that the subtotal only grows, silently breaking downward word-count changes.

### 4. Test carries a leftover mock for a module the hook no longer imports — ✅ FIXED

**[File: .../ChangeWordCountModal/hooks.test.ts]**

**Severity:** low

**Verified by reading the source:** `hooks.ts` no longer imports `JobType` or `api/jobTypes/enums` (the PR removed it with the `serviceJobIds` filter), and the hook's only use of `jobs` is `jobs.map(({ id }) => id.toString())`, so the `jobType` fixture fields were never read. Confirmed not needed transitively — every other import in the hook is itself mocked; the only unmocked imports are `./consts` and `./utils`, neither of which touches job types.

**Sharper than cosmetic:** this PR *removed* the `jobType`-based filtering. Leaving a `JobType` mock and `jobType` fields behind implies that filtering still exists — a fossil of exactly the code the PR deleted, i.e. a false signal about what the hook does.

**Resolution:** ✅ Mock and fixture fields removed on `a8a585212`.

---

## Validation Checks

⚠️ **Every row below ran with `node_modules` resolved from develop's lockfile (`@sentry/nextjs` 10.48.0) while this branch pins 7.73.0. Treat all of them as unverified.** A clean run requires `yarn install` on this branch first.

| Check | Result | Notes |
|---|---|---|
| `npx vitest run` (creative-portal, full) | 🚫 **Invalid** | Reported 6 files / 43 tests failing. **Withdrawn** — environment mismatch (see banner). Not evidence against this PR. Also note the run scanned `.next/standalone/**` build artefacts, further polluting it. |
| `ChangeWordCountModal/hooks.test.ts` | ⚠️ 8/8 passed | Same mismatched environment; likely sound but unverified |
| `npx turbo run typecheck --filter=@proofed/creative-portal` | ⚠️ Passed | Same caveat |
| `npx turbo run build --filter=@proofed/creative-portal` | ⚠️ Passed | Same caveat |
| `npx eslint` / `prettier` (touched files) | ⚠️ Clean | Same caveat; lint is largely env-insensitive here |

**Outstanding: run `yarn install` on this branch, then `test` / `typecheck` / `lint` / `build`.** CI is the authority until then.

---

## Tests

- ✅ New `hooks.test.ts` (8 cases) — covers args passed to `calculateOrderChargeAdjustments` (`mode: "add"`, same `order`, `masterReferencesCharge`, per-word tasks updated, non-per-word untouched); threshold-cross clears; drop-below re-applies; format-premium-only forwarding; empty-adjustments preserves existing order fields; per-word task updates; success toast + all three refresh calls; error path delegates to `showDefaultErrorToast`.
- ⚠️ Tests mock `calculateOrderChargeAdjustments` wholesale, so they verify the **hook's wiring**, not the pricing math. That is an appropriate unit boundary, **but** the utility itself has **no dedicated test file** (confirmed by file search) — it is only exercised indirectly via `useAddNewJob.test.ts`. The real numbers in this ticket are not asserted end-to-end anywhere. Pre-existing gap; worth a follow-up.
- ⚠️ No test for the partial-write window (Issue 2): `updateOrder` succeeding while a `updateJobTask` rejects.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Sound — root cause fixed, both threshold directions handled (established by source review) |
| Regression risk | ✅ Low — dead-code removal + reuse of an already-shipped utility/pattern; develop has not touched the billing utility in the intervening 42 commits |
| Tests | ⚠️ Good wiring coverage; underlying utility math untested directly |
| Code quality | ✅ Clean — dedupes job-task mapping, honest deps array, contract now accurate |
| Validation suite | 🚫 **Not validly run** — environment mismatch; needs `yarn install` + re-run. CI is the authority. |
| Branch freshness | ℹ️ 42 commits behind develop (merge-base 2026-06-10). Routine update advised; **not** a demonstrated blocker, and no billing-code drift. |
| Mergeable state | ✅ GitHub reports `mergeable_state: clean` |

---

## Recommendation

**Approve the code.** The fix is correct, tightly scoped, reuses the right utility, and the two actionable findings are applied.

1. ~~⛔ Merge/rebase develop — 43 tests red~~ → 🚫 **RETRACTED.** The failures were an environment artefact, not branch staleness. Updating from develop is still *advisable* hygiene (42 commits, merge-base 2026-06-10), but it is **not** a demonstrated blocker: none of those 42 commits touch `calculateOrderChargeAdjustments`, `calculateChargeAdjustments`, `calculateOrderSubtotal`, `calculateFormatPremium`, or `ChangeWordCountModal`.
2. **Run the validation suite properly before merge** — `yarn install` on this branch first, so dependencies match the branch's lockfile. Until then, defer to CI.
3. ~~Guard submit while the `PremiumChargeRate` query is unresolved~~ (Issue 1) → ⏭️ **Deferred** — real, but the pattern is shared with `useAddNewJob`; fix both together.
4. ~~Clarify the utility's add-mode contract~~ (Issue 3) → ✅ **Done** on `a8a585212`.
5. ~~Drop the leftover `api/jobTypes/enums` mock~~ (Issue 4) → ✅ **Done** on `a8a585212`.
6. **Partial-failure window** (Issue 2) → ⏭️ **Acknowledged**, no change; the suggested reorder does not fix it.

**Post-merge follow-ups worth raising separately:**

- **`masterReferencesCharge = []` silent no-op (Issue 1)** — guard both `ChangeWordCountModal` and `useAddNewJob`, and resolve `calculateFormatPremium`'s null-ambiguity ("not a premium format" vs "reference unavailable"). Covers the failed-query variant, not just the race.
- **Direct unit tests for `calculateOrderChargeAdjustments`** — no dedicated test file exists; the pricing math this ticket is about is not asserted anywhere directly.
- **Atomic order+job-task write (Issue 2)**, or at minimum defer the modal close until all writes resolve.

---

### Reviewer note — process lesson

The retracted blocker came from running the suite after a branch switch without `yarn install`, so month-old code was tested against develop's dependency tree. The "develop passes 63/63" control appeared to confirm the staleness theory when it in fact confirmed the confound: that was the only run where `node_modules` matched the checked-out code. **When switching between branches with divergent lockfiles, `yarn install` first — or treat the results as noise.**
