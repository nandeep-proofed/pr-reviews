# PR Review: feature/PP-2076: Customer Portal — support AI jobs (Pre/Post-edit) in order creation

**PR:** https://github.com/Proofed/B2BWebserver/pull/2429
**Jira:** https://proofed.atlassian.net/browse/PP-2076
**Status:** Blocked (Jira) — PR open, mergeable state clean, no CI checks configured, no human review yet
**Branch:** `feature/PP-2076-customer-portal-ai-jobs` @ `b87f47972` · 9 files, +491/−28
**Method:** 5 parallel finder lenses → merge/dedupe → 5 adversarial verifiers. 5 candidate findings were **refuted and dropped**; what remains was confirmed by a verifier that opened the cited lines itself, and in several cases by executing mutations or recomputing the arithmetic.

---

## What this means for users (non-technical summary)

1. **AI orders are created with wrong job deadlines.** Every AI job in a workflow is given the *first* AI job's due time, and AI clean-up work that is meant to run *after* editing is scheduled as if it ran *first*. In the author's own test order, both "AI Post-edit" jobs were given a deadline four days **before** the editing job they are supposed to follow. Orders are created successfully — the schedule attached to them is wrong.
2. **A customer who could succeed by trying again is told not to bother.** When an order genuinely can't be scheduled, the new message says "retrying will produce the same result — contact support." For one of the failure causes that is false: the same order can fail one minute and succeed the next, because the promised delivery date shifts around weekends. Some customers will abandon an order that would have gone through, and raise a support ticket instead.
3. **The specific bug this ticket exists to fix is not covered by any test.** The fix itself works — it was verified by hand and by probe. But no automated test runs the fix through the code that was rejecting AI jobs, so nothing stops it regressing.
4. **Orders created by this change do not start.** Confirmed by the author in the Jira ticket: AI jobs sit in "In Queue" forever because nothing picks up machine-assigned work. This is a backend gap, **not** caused by this PR, and is why the ticket is Blocked — but it means the ticket's goal ("AI workflows can be used for real customer orders") is not met end to end by merging this alone.
5. Everything else found is code health with no user-visible symptom.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| 1.1 Order creation completes for any valid workflow containing an AI job | Zero-length return windows raised to a 1-minute floor where delivery-calc data enters the BFF (`raiseZeroLengthReturnWindows`), so the timing chain advances. Author verified orders 21350/21355/21356 on B2B test. | ✅ Addressed — but see Issues 1 & 2: the order is created with an incorrect schedule |
| 1.2 A zero-length turnaround must be accepted, not rejected; advance control stays enabled | Same floor. The delivery-calculation preview route shares the helper. | ✅ Addressed |
| 2. Document upload against an AI workflow, all platform formats | **No code change in this PR.** Author reports manual verification with PDF and DOCX. The prior block was presumably a downstream consequence of order creation failing. | ⚠️ Partial — nothing in the diff targets upload support; rests entirely on manual verification |
| 3. No silent failures — clear message at the point of failure | `JobTimingValidationError` → `ApiError(400, ORDER_JOB_TIMING_ERROR_MESSAGE)`. Verified end-to-end: the message reaches the customer's toast verbatim on **both** the streamed and non-streamed paths, with a correlation ID. | ✅ Addressed — but the copy is wrong for one path (Issue 3) |
| Validation rule 1: advance control never left silently disabled | The message surfaces at order submission, not at the Charge step where the advance control lives. | ⚠️ Needs product confirmation |
| Validation rule 2: GDRIVE / Off-Platform not enabled | Untouched. | ✅ Respected |

**Scope creep:** none. All 9 files serve the ticket.

**Correction to the PR description:** it states the change makes timing failures "a 400 rather than an unhandled 500". `JobTimingValidationError` already carried `readonly statusCode = 400` on `develop`, and `handleEndpointError`'s `statusCode` branch already mapped it. Verified byte-identical on `origin/develop`. The real delta is the **message text**, which is still a worthwhile change — the framing just overstates it.

---

## Architecture Analysis

The fix normalises delivery-calculation data at the BFF boundary rather than relaxing the shared timing validator — the right call, and consistent with PP-2047 rejecting the relax-the-check approach on the Creative Portal side. Putting the raise in `createOrderFunc` rather than the routes is also correct: order creation has two entry points and the streamed one is what a customer hits when attaching a document.

The one architectural misstep is subtler. `apps/creative-portal` has this same `ORDER_TO_JOBS` map with `AI: 0.5` in its **delivery-calculation display route**, where ordering is cosmetic. Its **order-creation** path (`api/orders/createNew/utils.ts`) contains no sort at all — it consumes the delivery-calc jobs in service order. This PR is the first to apply that display-layer heuristic to a path that **persists `maxReturnTime` to OMS**, which is what turns a cosmetic ranking into a data-correctness problem (Issue 1).

---

## Issues Found

### 1. AI Post-edit jobs are scheduled ahead of the work they are meant to follow

**[File: apps/customer-portal/api/utils/deliveryCalculation/getDeadlineWithOrderedJobs.ts:36]**

> **In plain terms:** Clean-up work that is supposed to happen *after* an order has been edited is being scheduled as if it happened first. On an order that goes AI prep → Editing → AI clean-up, the clean-up step is given a due time before the editing has even started.

**Function/Class:** `sortByJobsInOrder` / `ORDER_TO_JOBS`

**Severity:** high

**Confidence:** high (verified)

**Steps to reproduce:**

1. As a customer, start order creation against a workflow containing an AI **Post**-edit job (e.g. AI Pre-edit → Editing → AI Post-edit → Review).
2. Complete order creation.
3. Open the order in the admin/creative area and inspect each job's due time.
4. **Expected:** the AI Post-edit job's due time falls after the Editing job's.
5. **Actual:** the AI Post-edit job is placed at the front of the schedule, so its due time precedes the Editing job's.

**Problem:** `AI: 0.5` ranks *every* AI job ahead of Service. The sort key is `jobType` alone, and `DeliveryCalculationType["jobs"][number]` (`apps/customer-portal/api/deliveryCalculation/types.ts:3-12`) exposes only `{ duration, jobType, jobTypeId, maxReturnWindowsMinutes, returnWindowsMinutes }` — there is no field distinguishing Pre-edit from Post-edit. A single rank per job type cannot express a type that legitimately appears both before *and* after Service.

**Evidence:** `getDeadlineWithOrderedJobs.ts:35-46`:

```ts
const ORDER_TO_JOBS = {
  AI: 0.5,          // <- added by this PR
  Service: 1,
  Review: 2,
  Return: 3,
  Other: 4
};

return (
  (ORDER_TO_JOBS[a.jobType] || ORDER_TO_JOBS.Other) -
  (ORDER_TO_JOBS[b.jobType] || ORDER_TO_JOBS.Other)
);
```

The sorted array is what drives the chain: `getDeadlineWithOrderedJobs.ts:59-61` → `createOrder.ts:101-103` (an order-preserving `.filter`) → `calculateJobsReturnTime.ts:26-56`, which walks `jobs.forEach` accumulating `previousMaxReturn`. Array position *is* chain position; there is no re-sort or re-key in between. Trigger: any order whose workflow contains an AI Post-edit job.

**Impact:** Wrong `maxReturnTime` persisted to OMS for the post-edit job and, because the chain is cumulative, for everything after it. Creatives and QA see out-of-order due times; anything driving escalation or overdue warnings off `maxReturnTime` fires immediately.

**Fix:** Order by the workflow's own sequence rather than a per-`jobType` rank. `JobConfiguration` already carries `jobSequence` (`apps/customer-portal/api/utils/configurations/types.ts:78-86`) and is currently unused here. Alternatively preserve the delivery-calculation response order on the persistence path, as the Creative Portal's `createNew` path already does — confirm with the delivery-calc service owner whether that response is in workflow order before relying on it.

---

### 2. Every AI job in a workflow is saved with the first AI job's deadline

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/helpers/createOrderAndJobs.ts:191-193]**

> **In plain terms:** When an order contains more than one step of the same kind — and every AI workflow does — all of those steps are saved with the same due time, copied from the first one. The correct due times for the second and third steps are calculated and then thrown away.

**Function/Class:** `createOrderAndJobs`

**Severity:** high

**Confidence:** high (verified against real production-test data)

**Steps to reproduce:**

1. As a customer, create an order from a workflow with two or more AI jobs (e.g. AI Pre-edit → Editing → AI Post-edit → Review → AI Post-edit).
2. Retrieve the created order's jobs from the admin area or the OMS job-search endpoint.
3. **Expected:** each job has a distinct due time, increasing down the workflow.
4. **Actual:** all AI jobs share one identical due time — the first AI job's.

**Problem:** `jobTimings` genuinely contains one entry per delivery-calculation job, so an AI workflow produces three distinct entries for `jobTypeId = 16`. `.find()` returns index 0 for all three job configurations, silently discarding the other two.

**Evidence:** `createOrderAndJobs.ts:188-204`:

```ts
for (const jobConfig of jobConfigurationsWithServices) {
  const timing = jobTimings.find(
    (entry) => entry.jobTypeId === jobConfig.jobTypeId
  )!;

  const newOrder = await addJob({
    jobConfigurationID: jobConfig.id,
    requesterId,
    body: {
      orderId: order.id,
      returnWindowsMinutes: timing.returnWindowsMinutes,
      maxReturnTime: timing.maxReturnTime
    }
  });
```

The pre-flight guard at `createOrderAndJobs.ts:129-141` uses `.some(...)` — existence only, never count — so it passes. `configuredJobTypeIds` (`createOrder.ts:93-97`) filters unconfigured rows but does not deduplicate.

Confirmed against the real API response the author recorded in Jira comment 70049 for order 21356:

```
26264  AI Pre-edit    maxReturnTime 2026-08-12T06:43:46.642
26265  Editing        maxReturnTime 2026-08-16T08:43:46.642
26266  AI Post-edit   maxReturnTime 2026-08-12T06:43:46.642   <- same as 26264
26267  Review         maxReturnTime 2026-08-20T10:44:46.642
26268  AI Post-edit   maxReturnTime 2026-08-12T06:43:46.642   <- same as 26264
```

`calculateJobsReturnTime.ts:42-47` throws unless each step is strictly after the previous, so three identical timestamps can *never* be produced by the chain — they are the signature of the many-to-one lookup, not of the calculation.

This is **pre-existing code that this PR makes reachable**: before the change, `calculateJobsReturnTime.ts:42` threw on the first zero-window AI job, so no AI order could be created at all and the collision could not occur.

**Impact:** Two of three computed timings are discarded on every AI order this PR enables. Combined with Issue 1, AI Post-edit jobs are born due days before the editing they follow.

**Fix:** Pair on workflow sequence rather than job type. The Creative Portal already solved exactly this at `apps/creative-portal/api/orders/createNew/utils.ts:551-554`, whose comment names the case explicitly: *"`jobTimings` is produced 1:1 from `mergedDeliveryJobs` … so the index lookup is exact even if a workflow ever contains two jobs sharing a `jobTypeId`."*

⚠️ **Do not port that index lookup verbatim.** In the Creative Portal the two arrays are 1:1 by construction. Here `jobConfigurationsWithServices` (OMS order, carrying `jobSequence`) and the sorted `jobTimings` are two *different* orderings, so index-pairing would mis-pair. Pair on `jobSequence`, and stop re-sorting the delivery-calc rows on the persistence path (Issue 1) — the two fixes belong together.

**Note:** Issues 1 and 2 are independent causes of one symptom. Fixing only the sort still leaves the post-edit job taking the pre-edit's timing via `.find()`. Fixing only the pairing still leaves the post-edit row positioned before Service in the chain. Both are needed.

---

### 3. Customers are told not to retry an order that would succeed a minute later

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/consts.ts:12-13]**

> **In plain terms:** When an order can't be scheduled, we tell the customer that trying again is pointless and they should contact support. For one of the reasons this happens, that's untrue — the very same order can fail at one moment and go through sixty seconds later, because the promised delivery date shifts around weekends. Customers will abandon orders and open support tickets they didn't need to.

**Function/Class:** `ORDER_JOB_TIMING_ERROR_MESSAGE`

**Severity:** medium

**Confidence:** high (arithmetic independently recomputed)

**Steps to reproduce:**

1. Configure an order whose job windows sum close to the promised turnaround, on a delivery configuration with `weekendDelivery: false`.
2. Submit it at a moment where the weekend buffer does not yet apply.
3. **Expected:** either it succeeds, or the customer is told something they can act on truthfully.
4. **Actual:** it fails with "retrying will produce the same result. Please contact support" — and the same submission succeeds moments later once the weekend buffer applies.

**Problem:** The doc comment at `consts.ts:7-9` asserts *"Every path that raises it depends only on the delivery configuration and the delivery-calculation response, neither of which changes between attempts."* That holds for four of the five throw sites. It is false for `calculateJobsReturnTime.ts:72-83` ("Final job maxReturnTime exceeds order dueDateTime"), because the due date is derived from `new Date()` via `calculateWeekendAdjustedDelivery`, whose `weekendsCount` depends on the day and minute of the attempt.

**Evidence:** `consts.ts:12-13`:

```ts
export const ORDER_JOB_TIMING_ERROR_MESSAGE =
  "We couldn't schedule the jobs for this order. Please contact support — retrying will produce the same result.";
```

`calculateDeadlineFromDeliveryCalculation.ts:29-34` passes `startDate: new Date()` into `calculateWeekendAdjustedDelivery`; `weekendCalculations.ts:105-129` derives `weekendsCount` from `dayOfWeek` and `minuteOfDay`. Recomputed (TZ=UTC, `weekendDelivery: false`, `totalOrderDuration: 2880`, `Σ maxReturnWindowsMinutes: 4000`):

| attempt (UTC) | day | weekendsCount | slack | result |
| --- | --- | --- | --- | --- |
| 2026-08-26T12:00Z | Wed | 0 | 2880 | **throws** |
| 2026-08-27T12:00Z | Thu | 1 | 5760 | **succeeds** |

The boundary is minute-granular: `2026-08-27T00:00:00Z` throws and `2026-08-27T00:01:00Z` succeeds. `weekendDelivery` defaults to `false` throughout (`weekendCalculations.ts:81`, `useDeliveryCalculationsWithDocuments.ts:84`), so this is the common configuration, not an edge case.

**Impact:** Abandoned orders and avoidable support load, in the exact place AC 3 asked for a message the customer can act on.

**Fix:** Drop the retry claim, or branch on the cause — `error.details` already distinguishes the sites (`{ finalMaxReturnTime, orderDueDateTime }` for the due-date check vs `{ index, jobTypeId }` for the others):

```ts
export const ORDER_JOB_TIMING_ERROR_MESSAGE =
  "We couldn't schedule the jobs for this order. Please contact support and quote the error ID below.";
```

Correct the `consts.ts:7-9` comment too — it currently states an invariant the code contradicts.

---

### 4. Nothing tests the bug this PR exists to fix

**[File: apps/customer-portal/api/utils/deliveryCalculation/utils.test.ts (and the two sibling new test files)]**

> **In plain terms:** The fix works — it was checked by hand. But no automated test runs it through the part of the system that was rejecting these orders, so if someone changes it later, nothing will notice it broke.

**Function/Class:** test suite coverage of `raiseZeroLengthReturnWindows` ↔ `calculateJobsReturnTime`

**Severity:** high

**Confidence:** high (demonstrated by probe)

**How to spot it:** Code health, not user-reproducible. `utils.test.ts` exercises the helper in isolation; `getDeadlineWithOrderedJobs.test.ts` mocks `./getDeadline` and stops before any timing math; `createOrder.test.ts:19-21` mocks `calculateJobsReturnTime` out entirely. The real function and the real helper never meet in any test.

**Problem:** The defect was an *integration* failure between the two. The tests assert the helper returns `1` instead of `0`; nothing asserts that `1` is *sufficient* for the consumer that rejected `0`. Set `MINIMUM_RETURN_WINDOW_MINUTES` to a value that still fails the chain and every test in this PR stays green.

**Evidence:** Run directly against the branch, the missing two-line before/after: raw zero windows produce `Job at index 0 (jobTypeId=16) maxReturnTime is not after the previous job's maxReturnTime` — the exact message quoted as motivation at `utils.test.ts:25-30` — and raised windows return timings cleanly. That test does not exist. CLAUDE.md requires tests for new code; the helpers are tested, the fix is not.

**Impact:** No regression guard on the ticket's central behaviour.

**Fix:** One test, no mocking of the timing function:

```ts
it("lets an AI workflow through the real timing chain", () => {
  const jobs = [
    { jobTypeId: 16, maxReturnWindowsMinutes: 0, returnWindowsMinutes: 0 },
    { jobTypeId: 1, maxReturnWindowsMinutes: 60, returnWindowsMinutes: 60 }
  ].map(raiseZeroLengthReturnWindows);

  expect(() =>
    calculateJobsReturnTime({ jobs, orderDueDateTime, orderStartTime })
  ).not.toThrow();
});
```

Add the companion asserting the *un-raised* list throws, so the test documents the bug as well as the fix. A variant with two AI jobs sharing `jobTypeId: 16` would also have caught Issue 2 — the PR's fixtures only ever use one.

---

### 5. A test that passes whether or not the code under test exists

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/__tests__/createOrder.test.ts:115-123]**

> **In plain terms:** One of the new tests would still pass if the change it is meant to protect were deleted entirely. It gives false confidence.

**Function/Class:** `it("rejects with a 400 rather than an unhandled 500")`

**Severity:** low

**Confidence:** high (mutation-proven)

**How to spot it:** Code health. Delete the whole `catch` block at `createOrder.ts:128-135`, replacing it with a bare rethrow, and run the file: this test still passes while its four siblings fail.

**Problem:** `jobTimingValidationError.ts:4` declares `readonly statusCode: number = 400`, so the raw error the mock throws already satisfies `toMatchObject({ statusCode: 400 })`. The `500` in the name never occurs on this path — `handleEndpointError.ts:37` matches `statusCode` before the 500 default, and that file is byte-identical on `develop`.

**Impact:** Minimal — the sibling at `:125-131` (`toBeInstanceOf(ApiError)`) genuinely guards the conversion. The test is redundant and its name misdescribes the change.

**Fix:** Merge into the sibling, asserting `toBeInstanceOf(ApiError)` *and* `statusCode: 400` in one test, and rename to what it checks.

---

### 6. Two test names describe behaviour they do not test

**[File: apps/customer-portal/api/utils/deliveryCalculation/getDeadlineWithOrderedJobs.test.ts:99]**

> **In plain terms:** Code health. A test is named as though it proves something it never checks.

**Function/Class:** `it("leaves non-AI jobs' windows untouched")`

**Severity:** low

**Confidence:** high (mutation-proven)

**How to spot it:** The test feeds only non-zero windows (60, 30), so it cannot distinguish AI-scoped from type-agnostic behaviour. Scoping `raiseZeroLengthReturnWindows` to `jobType === "AI"` leaves it green — only `utils.test.ts:72-83` fails, and that test in fact demonstrates the *opposite*: a `JobType.SERVICE` job with a zero window **is** raised.

**Problem:** Name/behaviour mismatch only. The type-agnostic behaviour itself is deliberate — PP-2047's commit `ec794749b` states *"No AI-specific branching: any job type with no scheduled duration is handled."* — so this is not a scoping defect, just a misleading label.

**Impact:** A future reader trusts a guarantee that isn't there.

**Fix:** Rename to `"leaves jobs with real windows untouched"`, and add a line to `utils.ts` noting the raise is intentionally type-agnostic.

---

### 7. All five timing failure causes collapse into one Sentry issue

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/createOrder.ts:128-136]**

> **In plain terms:** Code health. When scheduling fails, our error dashboard now shows one indistinguishable entry instead of telling us which of five causes fired.

**Function/Class:** `createOrder` catch block

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, no user symptom. `ApiError` takes `(statusCode, message)` only — no `cause` — and `handleEndpointError.ts:59` reports the *converted* error, so `reportError` never sees the original's message, `details`, or throw-site stack.

**Problem:** Five distinct causes (four in `calculateJobsReturnTime.ts`, one in `createOrderAndJobs.ts:137`) now arrive as one `ApiError` with identical message and stack, grouping into a single Sentry issue.

**Impact:** Lower than it first appears, which is why this is low rather than high: Logtail *does* preserve the full diagnostic. `@logtail/next`'s `jsonFriendlyErrorReplacer` explicitly re-adds the non-enumerable `name`/`message`/`stack`, so `logger.warn(..., { error })` records message, stack and `details` — and `handleEndpointError.ts:61-65` stamps the same `corr_id` the customer sees, so the two are one lookup apart. This is triage ergonomics, not blindness.

**Fix:** Optional. Either report the original before converting, or add an optional `cause` to `ApiError` and have `handleEndpointError` prefer it — the second helps every other `ApiError` conversion in the codebase.

---

### 8. The minimum-window helper will exist twice once PP-2047 lands

**[File: apps/customer-portal/api/utils/deliveryCalculation/utils.ts:18-19]**

> **In plain terms:** Code health. The same small rule is about to be written in two places in two apps, already with slightly different shapes.

**Function/Class:** `applyMinimumReturnWindow` / `MINIMUM_RETURN_WINDOW_MINUTES`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health. This PR adds one copy; `origin/fix/PP-2047-ai-job-zero-return-window` adds another at `apps/creative-portal/contexts/createOrder/utils.ts:13-16` with near-verbatim JSDoc and a wider signature (`number | null | undefined` vs `number`).

**Problem:** One business rule, two homes, already divergent in nullability. `packages/shared/utils/calculateJobsTime/` is the natural home — it owns `calculateJobsReturnTime` (the check this exists to satisfy), is consumed by both apps, and already declares the neutral shape `JobReturnTimingInput { jobTypeId, maxReturnWindowsMinutes, returnWindowsMinutes }` in its `types.ts:7-11`.

**Note on severity:** deliberately low, and the PR is not in violation of anything today — the second copy is on an unmerged branch, and the CLAUDE.md "reuse-first" rules as written cover components and pricing math, not this. The author already flagged the consolidation in the PR description. Only `applyMinimumReturnWindow` is genuinely shareable; `raiseZeroLengthReturnWindows` is typed on an app-local API type.

**Fix:** Whichever of PP-2076/PP-2047 merges second should delete its copy and consume a shared one, using the wider `number | null | undefined` signature so the form path works unchanged.

---

## Open Questions

- Does the delivery-calculation service return `jobs` in workflow order? If it does, the sort at `getDeadlineWithOrderedJobs.ts:59` is actively destroying correct ordering, and Issue 1's fix is simply to stop sorting on the persistence path. The Creative Portal's creation path trusts the unsorted order, but the PR's own test fixture assumes it is arbitrary. — `apps/customer-portal/api/utils/deliveryCalculation/getDeadlineWithOrderedJobs.ts:59`
- AC 2 (document upload for AI workflows) has no corresponding code change. Was the upload block purely a downstream consequence of order creation failing, or is there a separate gate that happens to be satisfied already? — no file
- Validation rule 1 asks that the advance control is never silently disabled. The new message appears at order submission, not at the Charge step. Is that the "point of failure" the ticket intended? — `apps/customer-portal/api/utils/mixtures/orders/createOrder/consts.ts:12`
- Should AI services count toward the format-premium gate? `createOrder/utils.ts:240-243` filters chargeable services to `jobTypeName === JobType.SERVICE`, while `calculateOrderSubtotalAtCreation` includes all chargeable services. A chargeable AI service would contribute to the subtotal but be invisible to the premium gate. Pre-existing filter, newly meaningful. — `apps/customer-portal/api/utils/mixtures/orders/createOrder/utils.ts:241`

---

## Pre-existing issues noted (not caused by this PR, not blocking it)

Worth separate tickets; several were surfaced only because this PR made their code paths reachable.

- **`weekendCalculations.ts:98-110` reads `getUTCDay()`/`getUTCHours()` off the result of `toZonedTime`.** The comment claims this works "regardless of the server's local timezone" — false for date-fns-tz v3. Verified: identical inputs give `weekendsCount: 1` under `TZ=UTC` and `0` under `TZ=Asia/Calcutta`, i.e. different promised delivery dates on a non-UTC server or any developer machine.
- **Temp-file leak on the streamed failure path.** `fileUploadStream.ts:348-371` runs `processed?.cleanup?.()` only on success; a throw from `businessLogic` skips it. This PR makes a timing rejection a routine, expected outcome on that path, so the leak now has a regular trigger — uploaded customer documents persist in the OS temp directory.
- **No rollback if job creation fails midway.** `createOrderAndJobs.ts:188-231` writes the order, then creates jobs in a loop, then patches to `Live`. A failure inside the loop leaves an orphan non-Live order with a partial job set.
- **Seven copies of the `ORDER_TO_JOBS` map, which disagree.** Two rank QA before Return, four rank Return before QA. The live inconsistency is between the WYSIWYG document builders and everything else. This PR authored none of them and only added `AI: 0.5` to one.
- **Grouped orders silently discard the weekend buffer.** The group anchor `deadline − totalOrderDuration` cancels exactly the weekend slack a non-grouped order keeps, giving a grouped order a materially narrower timing budget for identical work.
- **`vitest.config.ts` coverage `include` omits `api/**`.** All production code in this PR is invisible to coverage. Nothing is gated on it (no thresholds, no script, no CI reference), so it is informational.
- **`ApiErrorResponse` does not describe what `handleEndpointError` emits** — declared as an object, emitted as a bare string. Works because the create-order client guards with `typeof === "string"`, but the type misleads.

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ Skipped | Skipped at user request (workspace state — see below) |
| `npx turbo run typecheck` | ⏭️ Skipped | Explicitly skipped at user request |
| `npx turbo run lint` | ⏭️ Skipped | Skipped at user request |
| `npx turbo run build` | ⏭️ Skipped | Skipped at user request |

Validation was not run. The reviewer's working tree was dirty and on another branch, an in-place checkout was therefore unsafe, and a fresh worktree would have required a `yarn install` that could not complete (`TIPTAP_PRO_TOKEN` unavailable / registry returning 403). **The PR has no CI checks configured either** — `get_check_runs` returns zero — so nothing has verified this branch mechanically. Re-run `test`, `typecheck`, `lint` and `build` for `@proofed/customer-portal` before merging.

One partial signal: the three new test files were executed in isolation during review and **46/46 tests passed** across them and the four pre-existing `api/utils` test files, with no regressions. That is not a substitute for the full suite.

---

## Tests

- ✅ All three new test files run and pass (46/46 with neighbours); mocks are correctly wired and not vacuous — 6 of 8 targeted mutations were killed
- ✅ Every new production line has at least one test that fails when it is broken
- ✅ Mock paths all resolve to real modules with matching export shapes; `vi.mock` hoisting and `await import()` ordering are correct
- ✅ Vitest `include` and path aliases resolve all three files
- ❌ **No test drives the fix through the real `calculateJobsReturnTime`** — the single most valuable test is missing (Issue 4)
- ❌ No test uses more than one AI job, though every shipping AI workflow has two or three — this is why Issue 2 went unnoticed
- ❌ One test passes with the code under test deleted (Issue 5); two test names misdescribe what they assert (Issues 5, 6)
- ⚠️ `undefined`/`null`/`NaN` pass-through is documented in `utils.ts:14-16` but untested, and is reachable — per-job windows are never runtime-validated (`getDeadline.ts:30-38` checks only `totalOrderDuration`)
- ⚠️ Arguments to `calculateJobsReturnTime` are never asserted, so the `configuredJobTypeIds` filter and the grouped-order start anchor both survive deletion — pre-existing lines, but this file is their natural home

### Suggested manual QA script

1. **(Issues 1 & 2)** Create a Customer Portal order from a workflow with AI Pre-edit → Editing → AI Post-edit → Review → AI Post-edit. Open the order in the admin area and list every job's due time. Each should be distinct and increase down the workflow. Today all three AI jobs share one due time, and it precedes the Editing job's.
2. **(Issue 2)** Repeat with a workflow containing two Service jobs, or two Reviews — the same duplicate-type problem should be checked beyond AI.
3. **(AC 1.1)** Create orders from an AI-Pre-edit-only workflow and an AI-Post-edit-only workflow; both should complete through the Charge step.
4. **(AC 2)** Upload an on-platform document — DOCX, PDF, and at least one other platform format — against an AI workflow during order creation.
5. **(AC 3)** Force a scheduling failure and confirm the customer sees the plain-language message plus an error ID, and that the raw internal text (job indexes, `jobTypeId`) never appears.
6. **(Issue 3)** With the same failing order, retry after the weekend boundary and record whether it succeeds. If it does, the "retrying will produce the same result" copy must change before release.
7. **(Ticket blocker, not this PR)** After creating an AI order, confirm whether the AI jobs leave `In Queue`. Per Jira comment 70049 they do not — the order is Live but stalled. Confirm the backend ticket is tracked before this ships to customers.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ Fix works; the schedule it produces is wrong (Issues 1, 2) |
| Regression risk | ✅ Low — `JobType` enum addition breaks no consumer (no `Record<JobType,…>`, no exhaustive switch, no `Object.values`); no existing test asserts the old behaviour; the client discards `jobs` and reads only `totalOrderDuration` |
| Tests | ⚠️ Well-built but miss the seam the bug lived in |
| Accessibility | n/a — no UI change |
| Error handling | ✅ Message verified end-to-end to the toast on both routes; ⚠️ copy inaccurate for one cause |
| Security | ✅ No new request surface; no secrets; the change *reduces* internal-detail disclosure to customers. `/security` still required per CLAUDE.md |
| Code quality | ✅ Good — Prettier/import-order/naming clean, sound placement, genuinely well-documented; the type-safety of the new map is the best of the seven copies in the repo |
| Validation suite | ⏭️ Not run (user opted out; no CI on the PR either) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes.**

The approach is right, the diagnosis in the PR description is accurate and unusually well-evidenced, and the message plumbing is verified working end-to-end. But the change enables a path that persists an incorrect schedule on every order it creates, and the two causes are independent — both need fixing together.

1. **Fix Issues 1 and 2 together.** Pair jobs to timings by workflow sequence (`jobSequence`), and stop re-sorting delivery-calc rows on the persistence path. Do not port the Creative Portal's index lookup verbatim — the two arrays are not 1:1 here. Confirm with the delivery-calc service owner whether the response is already in workflow order.
2. **Add the missing integration test** (Issue 4) — the raise through the real `calculateJobsReturnTime`, plus a companion asserting the un-raised list throws, plus a fixture with two AI jobs sharing `jobTypeId`.
3. **Fix the retry copy** (Issue 3) and correct the `consts.ts` comment that states the false invariant.
4. **Tidy the three test-naming issues** (Issues 5, 6) — cheap, and two of the names currently overstate what the PR changed.
5. **Correct the PR description**: the 400 status was already the behaviour on `develop`; the real improvement is the message text.
6. **Run the validation suite** — nothing has verified this branch, in CI or locally.
7. **Run `/security`** per CLAUDE.md before merge.
8. Optional / follow-up tickets: Issues 7 and 8, plus the pre-existing items listed above — the `weekendCalculations` UTC bug and the temp-file leak are the two worth filing promptly.

**Note on the ticket:** PP-2076 is `Blocked` because AI jobs created via `CPOrderService` never activate. That is a backend gap, correctly diagnosed by the author and outside this PR. Merging this does not make AI workflows usable end-to-end on its own.
