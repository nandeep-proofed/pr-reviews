# PR Review: fix/PP-2047: accept zero-length return windows on AI jobs

**PR:** https://github.com/Proofed/B2BWebserver/pull/2415
**Jira:** https://proofed.atlassian.net/browse/PP-2047
**Status:** Code Review
**Branch:** `fix/PP-2047-ai-job-zero-return-window` → `develop`
**Diff reviewed:** 11 files, +429 / −121 (base `e5e016ee`, head `51a04b71`)
**Sibling PR:** [#2429 / PP-2076](https://github.com/Proofed/B2BWebserver/pull/2429) — Customer Portal half, open, same author

**Method:** six parallel finder agents (correctness, regressions/API contract, React/performance, reuse/conventions, test quality, error-handling/observability/security), then adversarial verifier agents tasked with *refuting* each severity-driving finding. Four candidate findings were dropped or downgraded by verification; one — "the Customer Portal is unguarded" — was **withdrawn entirely** after PR #2429 was found. Validation suite not run (user opted out).

---

## What this means for users (non-technical summary)

1. **Admins can create orders from AI workflows again.** The dead "next" arrow on the Charge step is fixed, and an AI step's status now displays instead of showing an empty box. This is the core of the ticket and it works.
2. **On a workflow with more than one AI step, the wrong jobs get built.** The default test workflow has three AI steps. On submit the first one is created three times and the other two are never created at all — with no error. This is a known fault the team consciously parked; this branch is what makes it start happening.
3. **If an admin marks one AI step as "Skipped", the remaining AI step loses its turnaround time** and the order is rejected at the final submit.
4. **Anyone who was already stuck on this bug before the fix ships stays stuck** — worse, they now get all the way to the end and hit a failure screen whose "try again" button can never succeed.
5. **A step can vanish from the Workflow screen** when a skipped step sits either side of an active one. Pre-existing, but this branch makes it much easier to hit.
6. **The "no silent dead-ends" promise in the ticket is still unmet.** One cause was removed; a negative turnaround value still produces the same dead button with no explanation.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| ---------------- | ----------------- | ------ |
| **1.1** Order creation completes for any workflow containing an AI job, across **all** on-platform channels | Creative Portal delivered here. Customer Portal delivered in sibling PR **#2429 (PP-2076)**, which adds its own `applyMinimumReturnWindow` in the customer BFF. Neither is merged. | ⚠️ Partial — split across two open PRs (see Issue 10) |
| **1.2** A zero-length turnaround must be accepted; advance control stays enabled; order creatable | `applyMinimumReturnWindow` raises `0` → `1` at hydration; `JOBS_SCHEMA` relaxed `min(1)` → `min(0)`. Verified the schema was the actual gate: `getCreateOrderSchema(CHARGE_STEP)` → `CHARGE_STEP_SCHEMA.orders` → `ORDER_SCHEMA.jobs` → `JOBS_SCHEMA`. | ✅ Addressed for new orders; ⚠️ not for pre-existing stuck forms (Issue 4) |
| **2.1** Document upload against an AI workflow, both portals | Not attempted here. PR #2429 reports it verified for the Customer Portal (PDF + DOCX). Creative Portal untested. | ⚠️ Partial |
| **3.1** No silent failures — a clear message at the point of failure | One trigger removed (twice over: the bound *and* the value that hit it). No error surface added. A negative window still reproduces the original symptom exactly. | ❌ Not met (Issue 6) |
| Not specific to AI Post-edit — reproduced on AI Pre-edit → Editing → Review (Jira comment 68868) | Handled: per-configuration keying, `hydratedJobIndexes`, `aiJobStatusOptions` extended. | ✅ Addressed |

**Undeclared scope.** The PR body describes 6 files, "creative-portal only". The branch has 11, and the additions it does not mention — the `mergeJobsByType` rewrite with a new `jobSequence` sort, the delivery-calculation payload rewrite, the `hydratedJobIndexes` matcher, the AI `In Queue` status option, the React key changes — are the riskiest part of the change (Issue 9).

---

## Architecture Analysis

Four distinct changes, only one of which the description covers.

**1. The declared fix.** `applyMinimumReturnWindow` raises an exact `0` to `MINIMUM_RETURN_WINDOW_MINUTES` where delivery-calc data enters Formik, plus `min(1)` → `min(0)`. Applying the floor at the entry point rather than inside the shared timing util is the right call: it preserves the strictly-increasing chain contract at `calculateJobsReturnTime.ts:42` that the Job Due Date pickers and the bulk shift depend on. The floor genuinely reaches the server — `mergeJobOverrides.ts:77-80` accepts an override only when `maxReturnWindowsMinutes > 0`, so `1` replaces the delivery-calc `0`. The decision to leave negatives unraised is deliberate and documented.

**2. `mergeJobsByType` rewritten** to key per job configuration and sort by `jobSequence`. Correct and necessary — without it a workflow running two jobs of the same type collapsed into one row. The sort's inputs are safe: `getSelectedDeliveryConfiguration.ts` builds `serviceConfigurations` by iterating `jobConfigurations`, so the `.find()` at `provider.tsx:179-181` always resolves and `jobSequence` is always present.

**3. Delivery-calculation payload rewritten** to resolve tasks by `jobConfigurationId === job.jobId` instead of splitting the merged `", "` task name. Correct — the old `find` by task name returned the same configuration twice for duplicated jobs.

**4. `aiJobStatusOptions` gains In Queue.** A real fix, not cosmetic: the shared `Select` resolves its value through `findValue`, which returns `undefined` when no option carries the value, so an AI job hydrated to `In Queue` previously rendered an empty dropdown.

**The structural risk is #2's blast radius.** Splitting one merged AI row into three separate rows is correct in the form, but three places downstream still resolve jobs by *type* rather than by configuration — the server's job-creation loop, the server's override merge, and the sibling window-sync effect. The form is now more precise than the code consuming it. That mismatch is Issues 1, 2 and 3.

---

## Issues Found

### 1. Multi-AI workflows build the wrong jobs on submit

**[File: apps/creative-portal/api/orders/createNew/utils.ts]**

> **In plain terms:** The standard test workflow contains three AI steps. When an order is created from it, the first AI step is built three times over and the other two are never built at all — the order looks created, but its workflow is wrong. Nothing errors, so nobody finds out until someone inspects the order. The team already knew about this fault and parked it as harmless; this branch is what stops it being harmless.

**Function/Class:** createOrderBusinessLogic — the job-creation loop

**Severity:** high

**Confidence:** high (mechanism); medium (on which of two outcomes occurs — see Evidence)

**Steps to reproduce:**

1. As an admin, create an order on the default B2B-test workflow: AI Pre-edit → Editing → AI Post-edit → Review → AI Post-edit.
2. Complete the flow and submit.
3. **Expected:** five jobs, one per workflow step.
4. **Actual:** the three AI steps all resolve to the *first* AI job configuration — three jobs are created against it and the other two AI configurations get no job. Job tasks, assigned-user details and the per-job pay breakdown for all three come from the first configuration.

**Problem:** `jobConfigurationsToCreate` is filtered by `jobId`, so after this PR it contains all three AI configurations (before it contained one, because the form merged them). The loop then resolves each delivery-calc job back to a configuration by **`jobTypeId`**, first match — and AI Pre-edit and AI Post-edit share one `jobTypeId`.

**Evidence:** `apps/creative-portal/api/orders/createNew/utils.ts:232-239` builds the candidate list by `jobId`:

```typescript
const jobConfigurationsToCreate =
  jobConfigurationsWithServices.filter((jobConfig) =>
    createOrderData.order.jobs.some(
      (job) =>
        job.jobId === jobConfig.id &&
        job.status !== JobStatusTypes.SKIPPED
    )
  );
```

then `:548-559` resolves by type:

```typescript
for (const [
  index,
  deliveryCalculationJob
] of mergedDeliveryJobs.entries()) {
  const currentJob = jobConfigurationsToCreate.find(
    (jobConfig) =>
      jobConfig.jobTypeId === deliveryCalculationJob.jobTypeId
  );

  const jobDetails = createOrderData.order.jobs.find(
    (job) => job.jobId === currentJob?.id
  );
```

`currentJob` flows to `jobConfigurationId: currentJob?.id` (`:630`), to `createJobTasks({ jobConfig: currentJob })` (`:673-679`), and to `jobServiceConfigurations = currentJob?.serviceConfigurations` which feeds the pay breakdown. The comment at `:561-563` — *"the index lookup is exact even if a workflow ever contains two jobs sharing a `jobTypeId`"* — justifies only `jobTimings[index]` two lines below, not `currentJob` above it. The pre-flight guard at `:300-312` compares by `jobTypeId` too, so it passes vacuously.

`mergeJobOverrides.ts:53-63` collapses identically, so the first AI job's window is applied to all three delivery rows and an admin's edit to the second or third AI row is silently discarded.

Verified that base collapsed the configurations: base `mergeJobsByType` keyed `jobGroups[serviceConfiguration.jobTypeName]` and set `jobId` only inside the initialiser, so one AI form job carried the first configuration's id and only that configuration reached this loop.

The outcome depends on whether the delivery calculation returns one entry per job *configuration* or one aggregated entry per job *type*. Evidence favours per-configuration (the captured 200 in the PR body carries a per-entry `tasks` array of service-configuration ids; this PR's own `hydratedJobIndexes` fix only makes sense under it; PR #2429 asserts the creative symptom is "all three get the same `jobConfigurationId`"). Under that reading: duplicate creation as described. Under the aggregated reading: no duplicates, but the second and third AI configurations still get no job, and this PR's `hydratedJobIndexes` fix would be broken in the opposite direction.

**Impact:** Orders from any workflow with two jobs sharing a `jobTypeId` are built wrong server-side, silently. AI workflows are the immediate case; two Services or two Reviews would behave the same.

**Fix:** Neither file is in this PR's diff, and PR #2429 records this as a deliberate deferral:

> *"A workflow with several jobs sharing a `jobTypeId` (AI workflows have three) breaks job↔timing matching in mirror-image ways… Creative Portal gives all three the same `jobConfigurationId`… Low impact while AI jobs are instant… Left alone deliberately — fixing one portal only would make the two diverge."*

That assessment was made against the pre-PP-2047 world, where `createNew` only ever saw one AI configuration. **This branch is what changes the impact**, so the deferral decision needs re-making before it merges — either fix both portals together as the note intends, or confirm the duplicate-creation outcome is acceptable. Technically, resolve the configuration positionally the way `timing` already is, or carry `jobConfigurationId` through `mergeJobOverrides` so each delivery job knows its own configuration. Change the `:300-312` pre-flight to compare configuration ids, not `jobTypeId`.

---

### 2. Skipping one AI step strands the other

**[File: apps/creative-portal/contexts/createOrder/provider.tsx]**

> **In plain terms:** If an admin sets an earlier AI step to "Skipped" in a workflow with two AI steps, the step that is still running loses its turnaround time. The order is then rejected at the final submit with a technical error message. The control that lets the admin do this is one this branch adds.

**Function/Class:** CreateOrderProvider — the PP-1641 window hydration effect

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. As an admin, start an order on a workflow with two AI jobs (AI Pre-edit → Editing → AI Post-edit).
2. On the Workflow step set the **first** AI job to **Skipped** — now selectable, because this PR adds it to `aiJobStatusOptions`. Leave the second In Queue.
3. Advance to the end and submit.
4. **Expected:** the order is created; the live AI job carries the turnaround the delivery calculation returned for it.
5. **Actual:** submission fails with a toast reading *"Job at index N … maxReturnTime is not after the previous job's maxReturnTime"*, and the live AI job's due-date fields are blank on the Workflow step beforehand.

**Problem:** The payload that produces `deliveryJobs` excludes skipped jobs. The matcher that maps the response back does not. The skipped row wins `findIndex` and consumes the entry; the running row is never hydrated. The new `hydratedJobIndexes` set fixed the same-type collapse but not the status dimension of the same matching problem.

**Evidence:** the payload filter at `provider.tsx:286-291`:

```typescript
tasks: (order?.jobs ?? [])
  .filter(
    (job) =>
      !!job?.jobTaskTypeName &&
      job.status !== JobStatusTypes.SKIPPED
  )
```

against the matcher it has to line up with, `provider.tsx:558-572`:

```typescript
const hydratedJobIndexes = new Set<number>();

deliveryJobs.forEach((deliveryJob) => {
  const jobIndex = order.jobs.findIndex(
    (
      job: CreateOrderSchema["orders"][number]["jobs"][number],
      index: number
    ) =>
      job.jobTypeName === deliveryJob.jobType &&
      !hydratedJobIndexes.has(index)
  );
```

No status clause. Downstream, `buildChainedMaxReturnTimes` returns `undefined` for a skipped job so the write onto it is inert, and the live job falls through `validateOrderJobTimings.ts:69-82` to its `job.window` fallback — `minTaskDuration`, `0` for AI — which throws at `calculateJobsReturnTime.ts:42`.

**Impact:** The ticket's primary acceptance criterion fails in a configuration reachable through the UI, using a control this PR introduced.

**Fix:** Mirror the payload's filter in the matcher — `JobStatusTypes` is already imported at `provider.tsx:21`:

```typescript
const jobIndex = order.jobs.findIndex(
  (
    job: CreateOrderSchema["orders"][number]["jobs"][number],
    index: number
  ) =>
    job.jobTypeName === deliveryJob.jobType &&
    job.status !== JobStatusTypes.SKIPPED &&
    !hydratedJobIndexes.has(index)
);
```

Better: extract "the jobs the delivery calculation was asked about" into one exported predicate used by both, so they cannot drift again.

---

### 3. A form saved before this deploy edits the wrong jobs after it

**[File: apps/creative-portal/contexts/createOrder/provider.tsx]**

> **In plain terms:** The new-order form saves itself as you work and restores when you come back — including across a deploy. A batch started before this change is restored with the old step layout, but any order added afterwards gets the new one. Editing a step's status or due date then writes to a different step in each order, and the order is submitted with those wrong values.

**Function/Class:** CreateOrderProvider — the `sharedOrder.jobs` initialiser

**Severity:** high

**Confidence:** medium-high (mechanism verified; depends on a user holding a saved form across the deploy)

**Steps to reproduce:**

1. Before the deploy, start a New Order batch on a workflow containing two AI jobs and leave it (close the tab — do not confirm the delete modal).
2. After the deploy, reopen New Order. The saved batch is restored with its old, merged job layout.
3. Drop one more file to add an order — that order is built with the new per-configuration layout.
4. Go to the Workflow step and change any job's status or due date.
5. **Expected:** the edit applies to that job in every order.
6. **Actual:** the edit lands on a different job in the newly added order and in the shared row, because the two orders' job arrays no longer line up.

**Problem:** `sharedOrder.jobs` is generated once and guarded against regeneration, and the persisted blob is restored wholesale with no schema version. Every positional write in the Workflow step targets one `jobIndex` across all orders *and* `sharedOrder`.

**Evidence:** the guard at `provider.tsx:236-250`:

```typescript
useEffect(() => {
  if (isSuccessSelectedDeliveryConfiguration && !formik.values.sharedOrder?.jobs) {
    const serviceJobs = createServiceJobs(serviceConfigurations);
    setFieldValueRef.current("sharedOrder", { jobs: serviceJobs });
  }
}, [isSuccessSelectedDeliveryConfiguration]);
```

Persistence is real and unversioned — I verified the whole chain rather than trusting either agent, because two of them disagreed on it. `NewOrderForm/index.tsx:444` mounts `<FormikPersist formId="new-order-form" />`; `packages/shared/hooks/useFormPersistence.ts:30-33` does `setValues(await getFormData(formId))` on mount and debounce-saves the entire `values` object; `packages/shared/utils/indexedDB.ts` opens `FormStorage` v1, store `formData`, and `getFormData` returns `result?.values`. Nothing clears `"new-order-form"` — `Steps/index.tsx:98` clears only the unrelated `"files"` key.

New-shape orders are appended to old-shape ones at `AddOrdersStep/hooks.tsx:362` + `:535-538`. The positional writers that then straddle both shapes: `WorkflowWindowInputRow/index.tsx:53-70`, `jobStatus/hooks.ts:103-149`, `generateWorkflowComponents.tsx:107-119`; `jobIndex` itself comes from a single order at `useWorkflowData.ts:54-63`.

A second consequence on the same boundary: the rewritten payload resolves tasks strictly by `jobConfigurationId === job.jobId` (`provider.tsx:292-297`), but an old-shape row merged several configurations while storing only the first one's id — so a rehydrated form silently sends an incomplete task list and gets a wrong deadline.

**Impact:** Wrong status/window/assignee values submitted, with no error. Within a single session everything stays aligned — the arrays are produced by the same deterministic call — so this is strictly a persistence-boundary defect.

**Fix:** Version the persisted payload — bump `formId`, or store a schema version and drop/regenerate `jobs` on mismatch. That also fixes the pre-existing case where switching workflow mid-flow leaves `sharedOrder.jobs` on the old configuration.

---

### 4. Users already stuck on this bug hit a retry loop that can never succeed

**[File: apps/creative-portal/contexts/createOrder/provider.tsx]**

> **In plain terms:** The fix deliberately lets through forms that were saved while the bug was live, so those users can get past the screen that blocked them. But nothing repairs the bad value in those saved forms — so they now travel all the way to the end, fail there, and land on a "try again" button that fails again every time, forever.

**Function/Class:** CreateOrderProvider — the hydration write guards

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Before the fix, get blocked at the Charge step on an AI workflow (this saves a `0` window). Close the tab rather than deleting the order.
2. After the fix ships, reopen New Order — the saved batch is restored, and Charge now advances.
3. Complete the flow and submit.
4. **Expected:** the order is created.
5. **Actual:** the failure screen appears with *"Job at index 1 (jobTypeId=1) maxReturnTime is not after the previous job's maxReturnTime"*. Its retry button returns to the Workflow step, and every subsequent submit fails identically.

**Problem:** `min(0)` was relaxed specifically to admit a persisted `0`, but the hydration guards only write when the field is *absent*, so a stored `0` is never raised.

**Evidence:** `provider.tsx:597` and `:604`:

```typescript
if (formJob.returnWindowsMinutes == null) {
...
if (formJob.maxReturnWindowsMinutes == null) {
```

`applyMinimumReturnWindow` has exactly two call sites (`:583`, `:586`), both wrapping the delivery-calc value; neither touches a persisted one. The `job.window` fallback does not rescue it — `validateOrderJobTimings.ts:79` tests `typeof … === "number"` and `0` **is** a number, so the `0` passes straight through to the throw. `JobTimingValidationError extends Error`, so `showDefaultErrorToast` renders `error.message` verbatim. `Steps/index.tsx:169-179` puts the user on `ORDER_FAILED_STEP`, whose retry returns to the Workflow step where nothing has changed.

**Impact:** Narrow and transitional — the PR's *primary* cohort (anyone creating an AI order after the deploy) is fully fixed and is not affected. Only users holding a pre-fix saved form are, and only when the affected AI job is In Queue; a persisted `0` on a Skipped job is filtered out before validation and is harmless.

**Fix:** Heal the value rather than admitting it, which also lets `min(1)` stay as the schema's honest invariant:

```typescript
if (formJob.returnWindowsMinutes == null || formJob.returnWindowsMinutes === 0) {
```

Or normalise windows once in `onLoadSavedData`. Separately, map `JobTimingValidationError` to a human-readable message before it reaches the toast.

---

### 5. The other half of the same-type fix was left behind

**[File: apps/creative-portal/contexts/createOrder/provider.tsx]**

> **In plain terms:** There are two places that match calculated turnaround data onto workflow steps. This branch fixed one and left the other, so the two now disagree about which step is which. A separate bug in the same code means an instant step's duration is never written at all.

**Function/Class:** CreateOrderProvider — `getWindowsTime` and the window sync effect

**Severity:** medium

**Confidence:** high

**How it shows up:** Not reliably reproducible on its own today — with every AI window at `0` the mis-assignment is invisible. It is a correctness gap in exactly the behaviour this PR set out to fix, and it is one reason Issue 2 fails hard rather than softly.

**Problem:** `getWindowsTime` reduces the delivery-calc jobs into an object keyed by job type, so two same-type entries collapse — last wins — and the effect writes that value to the *first* form row of that type. Separately, the guard treats a legitimate `0` as absent, so an AI job's `window` is never written even when it should be.

**Evidence:** `provider.tsx:375-381`:

```typescript
orderDurationDetails.jobs.reduce(
  (accumulator, current) => ({
    ...accumulator,
    [current.jobType]: [
      current.returnWindowsMinutes ?? current.duration
    ]
  }),
  {}
)
```

`provider.tsx:484-488` — a plain first-match, with neither the `Set` nor a status filter:

```typescript
const jobIndex = orders[orderIndex]?.jobs?.findIndex(
  (
    job: CreateOrderSchema["orders"][number]["jobs"][number]
  ) => job.jobTypeName === jobType
);
```

and `:490-494`, where `windowTimes[0]` is a truthiness test:

```typescript
const hasWindowTimes =
  jobIndex !== -1 &&
  windowTimes &&
  Array.isArray(windowTimes) &&
  windowTimes[0];
```

**Impact:** `job.window` is the fallback `validateOrderJobTimings` uses and the value shown in the assignment modal, so a stale or mis-assigned `window` feeds the submit-time chain check.

**Fix:** Key `getWindowsTime` by job configuration (or by delivery-job position) and reuse the same `hydratedJobIndexes` + status filter. Change `windowTimes[0]` to `typeof windowTimes[0] === "number"` so `0` is not read as absent. Simpler still: fold the `window` write into the effect that already walks `deliveryJobs` and delete `getWindowsTime`.

---

### 6. The "no silent dead-ends" requirement is still unmet

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/CreateOrderContent/index.tsx]**

> **In plain terms:** The ticket asks that a user never be left with a greyed-out next button and no explanation. This branch removed one cause of that, but added nothing that explains a failure — and one value it deliberately leaves alone still produces the identical dead button.

**Function/Class:** CreateOrderContent — the Next button

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. As an admin, reach the Charge step on an order where the delivery calculation returned a **negative** return window for a job.
2. Click Next.
3. **Expected (req 3.1):** a clear message naming what is wrong.
4. **Actual:** the arrow greys out. Nothing is rendered anywhere; the button is `pointer-events: none` so there is not even a tooltip.

**Problem:** The button is bound to `formik.isValid` with no sibling error output, and no component renders errors for the `orders[].jobs[]` sub-object. `applyMinimumReturnWindow` raises only an exact `0` — deliberately, so `calculateJobsReturnTime` still rejects negatives — so a negative still trips `min(0)` in the very field this PR touched.

**Evidence:** `CreateOrderContent/index.tsx:98-103`:

```typescript
{navigationConfig.showNext && (
  <NextStepButton
    type="submit"
    disabled={formik.isValidating || !formik.isValid}
  />
)}
```

`NavButton/styles.ts:77-80` is `&:disabled { pointer-events: none; opacity: 0.5; }`. The Charge step renders only `<ChargeTable />`, whose step-level error surface is still commented out at `ChargeTable/index.tsx:138-143` behind `{/* TODO - validation after call with designer */}`. The only write to these paths, `NewOrderForm/index.tsx:129-132`, is never rendered — it surfaces as a submit-time toast on the last step.

Verification narrowed the residual class considerably: `window`, `minimumWorkTime` and `status` are always assigned in `mergeJobsByType` and cannot be missing; `jobTypeName`/`jobTaskTypeName` are contract-guaranteed; and the Charge table's editable cells *do* mark errors with a red border that self-clears on edit. The genuinely exposed cases are a **negative window**, a nullish `jobId`, and stale persisted state.

**Impact:** The ticket's own Validation Rule — *"The advance control is never left silently disabled"* — is not satisfied.

**Fix:** Not fair to block this PR on it: the architecture behind it (`validateOnChange: false`, submit-only revalidation, a designer-pending error block) predates this branch, and requirement 2.1 is equally undelivered. Record 3.1 and 2.1 as outstanding ACs with the two concrete places to close them — the commented-out block at `ChargeTable/index.tsx:138-143` and the unrendered `setFieldError` at `NewOrderForm/index.tsx:129`. The one thing worth doing *here* is deciding whether a negative window should surface a message rather than silently disabling.

---

### 7. The riskiest changes have no tests; the trivial helper has four

**[File: apps/creative-portal/contexts/createOrder/provider.tsx]**

> **In plain terms:** The parts of this change most likely to go wrong are the only parts with nothing testing them, so a future edit can break AI order creation again without anything failing.

**Function/Class:** CreateOrderProvider — hydration matcher and delivery-calculation payload

**Severity:** medium

**Confidence:** high

**How to spot it:** No test file anywhere imports `CreateOrderProvider`. Coverage, not behaviour.

**Problem:** `applyMinimumReturnWindow` — a one-line ternary — has four tests. The two changes carrying the regression risk have none: the `hydratedJobIndexes` matcher (`:558-572`, where Issue 2 lives) and the payload rewrite (`:286-314`, which changed task resolution for every workflow).

The existing tests are genuinely mutation-sensitive, which is worth saying — the `mergeJobsByType` fixture deliberately supplies configuration 105 before 103 so the `jobSequence` assertion is real, and the `CHARGE_STEP_SCHEMA` test fails if `min(1)` is restored (I traced the full composition to confirm nothing else in the fixture fails first). Three gaps:

- **The constant's value is unpinned.** `expect(applyMinimumReturnWindow(0)).toBe(MINIMUM_RETURN_WINDOW_MINUTES)` imports the same constant the implementation uses, so setting it to `0` — which collapses the function to identity and reinstates the bug — still passes.
- **The merge arithmetic is unasserted.** The `mergeJobsByType` tests project away everything but `jobTaskTypeName` and `jobId`, so the `+=` accumulation of `window` / `minimumWorkTime` this PR rewrote is uncovered.
- **Nothing asserts the fix actually unblocks the chain.** No test feeds a raised window through `validateOrderJobTimings` / `calculateJobsReturnTime`.

**Impact:** CLAUDE.md requires tests for new code, and the uncovered matcher is exactly where Issue 2 lives.

**Fix:** Extract the matcher and the payload builder into pure helpers in `contexts/createOrder/utils.ts` and unit-test them — a skipped AI job before a live one; two AI jobs each hydrated from their own entry; a duplicated job yielding distinct `serviceConfigurationId`s. Add `expect(MINIMUM_RETURN_WINDOW_MINUTES).toBeGreaterThan(0)`, assert a whole merged row, and add one `validateOrderJobTimings` case covering both directions.

---

### 8. A job's card disappears between two skipped jobs (pre-existing)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/utils/generateWorkflowComponents.tsx]**

> **In plain terms:** On the Workflow screen, if a skipped step sits either side of an active one, the active step's card vanishes — the admin cannot see it or set its due date. This is an existing fault, not one this branch created, but splitting AI steps into separate rows makes it much easier to run into.

**Function/Class:** generateWorkflowComponents

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. As an admin, open the Workflow step on a workflow of at least three jobs.
2. Set the first job to Skipped, leave the second active, set the third to Skipped.
3. **Expected:** an "add job" control plus a card for the active job.
4. **Actual:** only the "add job" controls render; the active job's card is gone. Both controls also list the same entries, since they share one mutated array.

**Problem:** `items` accumulates across the whole reduce and is never reset, so from the second skipped job onward `acc.pop()` removes whatever was pushed last — a `WorkflowBox`, not the previous button.

**Evidence:** `generateWorkflowComponents.tsx:56` declares `items` outside the reduce; `:123-133`:

```typescript
// Remove any existing NewWorkflowButton component before adding the new one
if (items.length > 1) {
  acc.pop();
}

acc.push(
  <NewWorkflowButton
    key={`new-workflow-${index}`}
    {...{ items }}
  />
);
```

Verification transcribed the reduce and executed it: `[skipped, active, skipped]` yields two `NewWorkflowButton`s and no box; the five-job AI workflow yields three buttons and **zero** boxes. The QA guard does not rescue it. `WorkflowJobsSection.tsx:56` returns the array directly, so the popped element is genuinely not rendered.

**Impact:** UI-only and recoverable — the job stays in the form values with its status intact, and un-skipping anything re-renders the card. While hidden, the admin loses that job's status select, Minimum Work Time and due-date rows.

**Fix:** **Do not attribute this to the PR's author** — the block is byte-identical on `develop`, the diff changed only the two `key` props, and it was already reachable there with three distinct job types (skip the AI row, skip Review — `allowDeletion: true` puts Skipped in the Review select). This branch widens the blast radius by making three separately-skippable AI rows possible. Pop only when the previous entry really is a button, or collect `items` in a first pass and emit one button. Worth its own ticket if out of scope here.

---

### 9. `mergeJobsByType` no longer merges by type, and its new doc contradicts itself

**[File: apps/creative-portal/services/orders/utils.ts]**

> **In plain terms:** A shared function's name now describes the opposite of what it does, and the explanation added above it contradicts the code two lines further down.

**Function/Class:** mergeJobsByType

**Severity:** medium

**Confidence:** high

**How to spot it:** `utils.ts:174` and the doc comment at `:161-173`. Code health.

**Problem:** The function groups per job configuration and explicitly avoids grouping by type — its own new comment says *"Grouping by job type would collapse jobs that share one… each job configuration is its own step"*. The name still says otherwise. The same comment also claims the result keeps *"the order the service configurations arrive in"*, while `:219-224` sorts by `jobSequence` — and the new test asserts the sorted order, so the doc is wrong.

Two unreachable fallbacks introduced alongside: `serviceConfiguration.jobConfigurationId ?? serviceConfiguration.jobTypeName` (`:191-193`) and `group.jobConfiguration?.jobSequence ?? 0` (`:222-223`). `jobConfigurationId` is a required `number` (`api/service-configuration/types.ts:12`), `jobSequence` is required on `JobConfiguration`, and `getSelectedDeliveryConfiguration.ts` builds the array by iterating job configurations, so neither can fire. Both encode wrong semantics if they ever did: falling back to `jobTypeName` reinstates the exact collapse this PR fixes, and would leave `jobId` `undefined` against `Yup.number().required()`.

**Impact:** The next reader starts from a false premise — which is how the original bug was written.

**Fix:** Rename to `mergeTasksByJobConfiguration` (one production call site, `:246`, plus a test import), correct the ordering sentence, and drop both fallbacks so the `Map` key narrows to `number`.

---

### 10. The PR description omits most of the diff, and the split with PP-2076 isn't recorded

**[File: (PR description)]**

> **In plain terms:** The write-up describes a small contained change, but the actual change is twice that size and rewrites the code deciding what steps a workflow has and in what order. It also doesn't mention that half of what the ticket asks for lives in a separate open pull request.

**Function/Class:** n/a

**Severity:** medium

**Confidence:** high

**How to spot it:** "Areas of Change" lists 6 files, "creative-portal only". GitHub reports 11. Process hygiene.

**Problem:** Undocumented: the `mergeJobsByType` rewrite and `jobSequence` sort (behavioural for **every** workflow), the payload rewrite, the `hydratedJobIndexes` matcher, `aiJobStatusOptions` gaining In Queue, and the React key changes. The riskiest change in the branch is the one the description doesn't mention. The Testing evidence ("4,411 passed", "build successful", "lint clean") predates those commits, and `## Results` appears twice.

Separately, the body says the customer portal *"does not run AI workflows, so it needs no backstop"* — written before PP-2076 existed. PR #2429 now delivers that half and duplicates `applyMinimumReturnWindow` into the customer BFF, with the author noting it is *"worth consolidating into `packages/shared` once PP-2047 lands."* That consolidation is a real follow-up and a merge-order dependency, and neither PR's description links the other.

**Impact:** A reviewer working from the description reviews the safe fraction. QA gets no signal to re-check a non-AI workflow against the new sort.

**Fix:** Update "Areas of Change" to the real diff; flag the `jobSequence` sort as affecting all workflows; cross-link #2429 and state the merge order and the `packages/shared` consolidation; re-run and re-state validation against head; drop the duplicate heading.

---

### 11. Smaller items

**[File: various]**

> **In plain terms:** A handful of small things — a duplicated dropdown definition, an unused export, two identical selectors on screen at once, and a comment pointing at the wrong function.

**Severity:** low · **Confidence:** high

- **`jobStatus/index.tsx:55`** — `instanceId={currentJob}` is the task name. Two rows sharing a name now emit duplicate DOM ids across react-select's input, listbox, `aria-controls` and `aria-describedby`. Use `` `${currentJob}-${currentlyEditedJob}` `` — the index is already passed in.
- **`WorkflowStep/consts.ts:66-71`** — `reviewJobStatusOptionsDemo` has zero import sites, while `generateWorkflowComponents.tsx:171-176` builds a byte-identical object inline. This PR de-duplicated the other two option lists in this file and missed the largest one. The `Demo` suffix is also misleading — these are production options.
- **`contexts/createOrder/consts.ts:11-13`** — the comment cites `getUpstreamJobConstraint`, which serves the post-creation order-management sidebar, not this flow. The check that actually rejects a zero window here is `calculateJobsReturnTime.ts:42`.
- **`contexts/createOrder/utils.ts:12-16`** — the signature accepts and returns `number | null | undefined`, but the only call site passes `deliveryJob.returnWindowsMinutes ?? deliveryJob.duration`, both non-optional `number`. The widened return erases the `number` guarantee downstream. Narrow it, or document that the tolerance exists because the response isn't runtime-validated.
- **`NewSelectAssignModal/index.tsx:244`** — reads `orders[0].jobs[0].window` regardless of which job is being assigned. The new sort changes which job sits at index 0, so on the AI workflow this caption now reads an AI job's zero duration while assigning Editing. Use the index already read at `:59-62`.
- **`services/orders/utils.ts:165` and `schemas.test.ts:101`** exceed the repo's 70-char Prettier width. Prettier doesn't reflow comments, so `yarn format` won't catch them.
- **`schemas.ts:32-33`** — `min(0)` without `.integer()`, while the server's own `putJobs/schema.ts:18-21` uses `.integer().min(0)`. A fractional window now passes and `addMinutes` does not truncate, so a `maxReturnTime` with seconds could persist.

---

## Withdrawn during verification

Recorded so the same ground isn't re-covered, and because two of these were in my own first-pass draft:

- **"The Customer Portal still rejects AI workflows"** — *withdrawn.* Asserted by three finders and my initial draft, all reading `develop` in isolation. PR **#2429 (PP-2076)** delivers it: `applyMinimumReturnWindow` in the customer BFF, applied in `getDeadlineWithOrderedJobs.ts`, which feeds the `createOrder.ts:94` call site. It also adds `AI: 0.5` to the customer ordering map, closing a separate open question about AI sorting after `Return`. What survives is only the merge-order/duplication note in Issue 10.
- **"The form isn't persisted, so `min(0)` guards nothing"** — *refuted.* One finder concluded IndexedDB holds only files. It missed the `FormikPersist` → `useFormPersistence` indirection. The full form round-trips; the PR's stated rationale for `min(0)` is correct.
- **"Requirement 3.1 is a blocker on this PR"** — *downgraded to medium.* The silent-disable architecture predates this branch, and requirement 2.1 is equally undelivered, so singling out 3.1 as blocking is inconsistent.
- **"The vanishing card is newly reachable"** — *corrected.* Already reachable on `develop` with three distinct job types. Pre-existing, widened here.
- **"AI jobs default to Skipped"** — *unsupported.* All 20 workflow templates set `requireCreation: true`, including "AI Package". Issue 8 needs a user to skip two non-adjacent jobs.
- **Query-key churn, React key instability, and the new `flatMap`'s complexity** — all checked and cleared with evidence (React Query hashes keys structurally; the sort runs once at form init; the replaced code had identical complexity).

---

## Open Questions

- Does the delivery calculation return one entry per job **configuration** or one aggregated entry per job **type**? This decides which of the two outcomes in Issue 1 occurs, and whether `hydratedJobIndexes` works at all. Evidence favours per-configuration but no captured multi-AI response exists in the repo. **This is the single most valuable thing to confirm before merging.**
- The BFF sorts the delivery response by job-type category at `api/deliveryCalculation/deliveryCalculation.ts:45-74`, but the comparator reads `a.jobType` — which the captured live response doesn't carry (`jobType` is derived client-side from `jobTypeId` in the provider's `select`). If the field is absent server-side every job scores `Other` and the sort is a no-op. Is that intended, and does the response then arrive in workflow order?
- Do two configurations sharing a `jobTaskTypeName` ever differ on `chargeable`/`payable`? `generateWorkflowComponents.tsx:71-73` still resolves them by name and takes the first match.
- `SKIPPED_JOB_STATUS_OPTION` / `IN_QUEUE_JOB_STATUS_OPTION` are now shared object references across three option lists. Does any `Select` consumer mutate an option in place?
- `apps/creative-portal/components/pages/admin-area/orders/partials/WorkflowStep/consts.ts` is an orphaned near-verbatim copy of the file this PR edits, with no importers. Should it be deleted?

---

## Validation Checks

| Check | Result | Notes |
| ----- | ------ | ----- |
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

Not run for this review. The PR body's evidence predates several commits on the branch — re-run all four against head `51a04b71`. The body also flags a pre-existing lint failure on `develop` (unused `isBefore` at `admin-area/orders/hooks.tsx:12`); that import is no longer at that location on the current branch, so re-confirm.

---

## Tests

- ✅ `contexts/createOrder/utils.test.ts` — covers 0, a real value, a negative, null/undefined. Catches a reverted implementation (but not a mutated constant — Issue 7).
- ✅ `schemas.test.ts` — the `CHARGE_STEP_SCHEMA` case reproduces the exact rejected shape and fails if `min(1)` is restored; composition traced end to end.
- ✅ `services/orders/utils.test.ts` — covers both the duplicate-job case and single-job task merging, with a deliberately out-of-order fixture that makes the sort assertion real.
- ✅ `WorkflowStep/consts.test.ts` — premise verified (`findValue` returns `undefined` for an unlisted value). Over-coupled: it also fails if `mergeJobsByType` regresses, so a failure points at the wrong file.
- ❌ No test for the `hydratedJobIndexes` matcher — the ticket's actual fix.
- ❌ No test for the delivery-calculation payload rewrite.
- ❌ No test for the React key change.
- ❌ No test that a raised window actually clears the timing chain.

### Suggested manual QA script

1. **(Issue 1)** Default workflow (AI Pre-edit → Editing → AI Post-edit → Review → AI Post-edit). Create an order, then inspect its jobs in the admin panel or via the jobs API. Five jobs must exist, one per step, with five distinct job configurations.
2. **(Issue 2)** Same workflow. Set the first AI job to Skipped, leave the others In Queue, submit. The order must be created with no error toast.
3. **(Issue 3)** Start a batch on the current build, leave it without deleting, deploy the branch, reopen, add one more file, then change a job's status on the Workflow step. The edit must apply to the same job in every order.
4. **(Issue 4)** With a pre-fix saved form holding a zero window on an In Queue AI job, complete the flow. It must submit — not land on the failure screen.
5. **(Issue 6)** Force a negative return window. A message must appear at the point of failure, not a greyed-out arrow.
6. **(Issue 8)** Skip the first and third jobs of a three-job workflow. The middle job's card must still render.
7. **(Regression)** A plain non-AI workflow (Editing → Review → QA): step order unchanged from before this branch, and the same total duration.
8. **(Req 1.2)** AI Post-edit workflow: Charge advances, and the AI job's status reads "In Queue" rather than an empty field.

---

## Summary

| Aspect | Status |
| ------ | ------ |
| Correctness | ⚠️ Right diagnosis, right fix for the form; three matching gaps remain downstream (Issues 1, 2, 5) |
| Regression risk | ❌ High — the `jobSequence` sort changes step ordering for all workflows, and Issues 1 and 3 are silent-corruption paths |
| Tests | ⚠️ Genuinely mutation-sensitive where they exist; absent on the two riskiest changes |
| Accessibility | ⚠️ Duplicate react-select `instanceId` for same-named jobs |
| Error handling | ⚠️ Req 3.1 unmet; submit failures render raw internal messages |
| Security | ✅ No new inputs, network surface or secrets; one bound relaxed with negatives still rejected. `/security` not run — required before merge |
| Code quality | ⚠️ Unusually clear comments and reasoning; `mergeJobsByType`'s name and doc now contradict its behaviour |
| Validation suite | ⏭️ Not run — re-run against head before merging |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes** — on two items, not eleven.

The fix itself is well-diagnosed and well-argued. The floor is applied at the right layer, the shared timing contract is deliberately preserved with the reasoning recorded, the `min(0)` rationale is documented in a test rather than a comment, and the `mergeJobsByType` rewrite is the correct root-cause fix rather than a patch. Most of what follows is downstream of that rewrite being *more* correct than the code consuming it.

Before merge:

1. **Issue 1** — settle the multi-AI job creation. The `.find()` by `jobTypeId` is pre-existing and PR #2429 records it as a deliberate joint deferral, but that call was made when the creative portal never sent two same-type configurations. This branch changes that, so re-make the decision: fix both portals together as #2429 intends, or confirm the duplicate-creation outcome is acceptable and say so explicitly. **First, answer the delivery-calc response-shape question in Open Questions** — it determines which outcome you're accepting.
2. **Issue 2** — add the `status !== SKIPPED` filter to the hydration matcher. One line, and it closes a path that fails the ticket's own primary AC.

Strongly recommended, not blocking:

3. **Issue 3** — version the persisted form, or regenerate `jobs` on a shape mismatch.
4. **Issue 7** — extract the matcher and payload builder as pure helpers and test them; that also makes Issue 2 impossible to reintroduce.
5. **Issue 10** — update the description to the real diff, cross-link #2429, and state the merge order and the `packages/shared` consolidation.
6. **Issues 4, 5, 9** — heal persisted zeros rather than admitting them; finish the same-type fix in the window-sync effect; rename `mergeJobsByType` and drop the dead fallbacks.

Track separately: **Issue 6** (req 3.1) and **Issue 8** (pre-existing, widened here) belong on their own tickets alongside the still-undelivered req 2.1 — neither should block this PR.

Finally: re-run `test` / `typecheck` / `lint` / `build` against head `51a04b71`, and run `/security` — validation was skipped for this review.
