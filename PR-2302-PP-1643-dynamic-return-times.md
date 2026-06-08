# PR Review: feature/PP-1643: Dynamic return times

**PR:** https://github.com/Proofed/B2BWebserver/pull/2302
**Jira:** https://proofed.atlassian.net/browse/PP-1643
**Status:** Code Review
**Scope:** 64 files changed, +2,966 / −528

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| **1.1** Unassigned card shows standard window + `maxReturnTime` as two pills | `JobReturnTimesTray` renders `TimingPill` user + max with divider per Figma 73633:28140 | ✅ Addressed |
| **1.2** No countdown on unassigned jobs | `useJobReturnTimesTray` clears countdown via the tray's `isFinishedOrder` gate; the left pill shows duration of the projected window, not a live countdown of `returnTime` | ✅ Addressed |
| **1.3** "At risk" badge when `now + returnWindowsMinutes > maxReturnTime` | `getAtRiskBufferMinutes` + `isAtRiskBuffer` flag; `BufferChip` flips to red. Tested in `index.test.tsx` "marks the buffer at-risk…" | ✅ Addressed |
| **2.1–2.3** User DD picker shoots between `now < pickedDateTime ≤ maxReturnTime` (unassigned only) | `DueDatePicker` (mode=user) + `useUpperBound` shifts `maxReturnTime` into picker frame; `disabled.after` matcher caps the calendar | ✅ Addressed |
| **2.4** On save: `returnWindowsMinutes = newReturnWindowsMinutes`, `maxReturnTime` unchanged, `returnTime = null` | `updateReturnWindow` PUTs only `{ id, orderId, returnWindowsMinutes }`; `mergeJobPutBody` preserves `maxReturnTime` from current state, omits `returnTime` when both incoming and current are undefined | ✅ Addressed |
| **3.1–3.3** Assigned card shows `returnTime`, countdown, overdue style when `now > returnTime` | `isAssigned(timing)` swaps to the green checked-user icon; `isOverdueLeft/Right` flips both pills red; countdown derived from `differenceInMinutes(leftDate, nowUtc)` when overdue | ✅ Addressed |
| **4.1** `returnTime == maxReturnTime` collapses to 0h 0m | `getProjectedReturnTime` clamps to `maxReturnTime`; `getAtRiskBufferMinutes` returns 0; `bufferText` is `0h 0m` | ✅ Addressed |
| **5.1** Shared component accepts nullable timings | `DeadlineDisplayProps.dateTime` widened to `string \| Date \| null \| undefined`; `formatZonedDateTimeWithDuration` returns empty-state when invalid | ✅ Addressed |
| **5.2** Renders fallback when missing rather than "Invalid Date" | `isNotEmptyDate` guard in `formatZonedDateTimeWithDuration`. **But** the `OrderStatusCardInfo/template.tsx` `DeadlineRow` shows an infinite loading spinner for unassigned jobs (see Issue 5) | ⚠️ Partial |
| **6.1** Effective deadline `returnTime ?? maxReturnTime` for tables | `jobItemToJobTableItem` uses `getEffectiveDeadline(parseJobTiming(jobItem))` → `returnTimeUtc ?? maxReturnUtc` | ✅ Addressed |
| **6.2** Visual distinction locked-deadline vs hard-stop | Two pills with distinct icons (User vs Calendar) + two-tone styling per Figma | ✅ Addressed |
| **Comment-55721 cascade** Editing Job DD can't push next unassigned job's User DD past its ceiling | Three-layer defense: DayPicker `disabled.after`, inline `jobPickerError` in Target mode, hook-level `getDownstreamConstraintErrorText` toast | ✅ Addressed |
| **Shift mode** Cascade downstream `maxReturnTime` by the delta | `updateMaxReturnTime` (mode = Shift) dispatches parallel PUTs for downstream jobs | ✅ Addressed |

**Beyond-Jira scope** (acknowledged in PR description):
- `mergeJobPutBody` helper + `putJob.ts` body-id ↔ URL-jobId guard (security hardening)
- `parseUtcDateString` shared util replacing scattered `new Date(\`${x}Z\`)` patterns
- `toHoursAndMinutes` -0 edge case + `alwaysShowMinutes` option
- Conventions sweep (stale styled exports, line-height:0, double-imports)

---

## Architecture Analysis

The PR cleanly separates the three timing fields into pure helpers (`parseJobTiming`, `getProjectedReturnTime`, `getDisplayedReference`, `getAtRiskBufferMinutes`, `getDownstreamJobConstraint`) co-located in `OrderJobs/utils.ts`. The presentational layer (`JobReturnTimesTray` + `TimingPill` + `BufferChip`) is hook-driven (`useJobReturnTimesTray`) — index.tsx is UI-only per CLAUDE.md.

The picker stack is well-designed: `DueDatePicker` wraps `DeadlineDatePicker` with a discriminated `UserModeProps | JobModeProps` union, and `useUpperBound` returns picker-frame ceilings while `useDueDatePicker` exposes label/mode state. The hook's frame contract (`toPickerFrame` / `fromPickerFrame`) is explicit and consistent across `JobCard` seeding and apply-path conversions.

`useUpdateJobReturn` flattens the previous lazy `buildBody` wrapper into a `runGuards` predicate runner that returns boolean — cleaner control flow.

Concerns:
- `JobCard.tsx` is now 737 lines with all picker render-prop closures and effect wiring inlined. The PR description acknowledges this as a deferred `useJobCard` extraction follow-up.
- The PR is the top of a stacked-PR sequence (PP-1640 → PP-1641 → PP-1642 → PP-1643). Helpers like `parseJobTiming` and `getDownstreamJobConstraint` live in this PR's `OrderJobs/utils.ts`, so PP-1643 is self-contained for the helpers it consumes.

---

## Issues Found

### 1. `updateReturnTime` sequence guard is silently dead — Z-suffix doubled on updated jobs

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturn.ts]**

**Function/Class:** `updateReturnTime` (lines 106–129)

**Severity:** high

**Problem:** `updateJobReturnTimes` (in `OrderJobs/utils.ts` line 199) emits new `returnTime` values via `.toISOString()` — already `Z`-suffixed (`"2026-05-18T18:00:00.000Z"`). The sequence-guard loop then does `new Date(\`${job.returnTime}Z\`)`, which appends another `Z` → `"2026-05-18T18:00:00.000ZZ"` → `Invalid Date`. `areDatesInSequence` compares `Invalid Date <= prev` (always `false`), so `!some(...)` returns `true` → the function reports "in sequence" for any input, silently bypassing the guard.

**Impact:** Cascade-mode updates that violate the sequence-by-prior-job rule no longer trigger the "due date cannot be earlier than the previous job's due date" toast. The `setDeadlineModalProps` warning below still fires (it uses `getBufferMinutes(order.dueDateTime, job.returnTime)` which uses `parseUtcDateString` correctly), but the sequence check itself is dead. Easy regression vector for any future cascade-cascade interaction.

**Fix:** Use `parseUtcDateString` (which handles both naive OMS strings and Z-suffixed `.toISOString()` output without double-appending):

```typescript
const updatedJobsReturnTime = [
  ...(previousReference
    ? [parseUtcDateString(previousReference)]
    : []),
  ...updatedJobs.flatMap((job) =>
    job.returnTime ? [parseUtcDateString(job.returnTime)] : []
  )
];
```

The previous-PR version (before HEAD) used `new Date(job.returnTime)` (no Z append) for updated jobs — that worked because `.toISOString()` is already a valid ISO string. The current code conflated the two shapes. A targeted unit test in `OrderJobs/utils.test.ts` for `areDatesInSequence` paired with `updateJobReturnTimes` output would catch the next regression.

### 2. Picker open date disagrees with tray pill date for jobs index > 0

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/partials/JobCard.tsx]**

**Function/Class:** `initialDeadlineDate` useMemo (lines 234–251)

**Severity:** medium

**Problem:** The tray's `useJobReturnTimesTray` hook anchors the projected return on `previousJobMaxReturnTime` (line 192 of JobCard.tsx → hook line 70 of `JobReturnTimesTray/hooks.ts`), so subsequent-job pills show "previousMax + window". But `initialDeadlineDate` in JobCard projects from `referenceNowUtc` (real now), not from `previousJob.maxReturnTime`. When the admin opens the User DD picker on a subsequent job, the picker pre-selects a different date than the pill displays.

**Impact:** UX inconsistency: pill says "Friday 4:20pm (4h 0m)" but the picker opens at Wednesday afternoon. Admin clicks the pill expecting "edit this date" and gets a stale starting position. The `onOpenChange` handler does `setDeadlineDate(initialDeadlineDate)` on every open, so this resets to the wrong anchor every time.

**Fix:** Project from the same anchor the tray uses:

```typescript
const previousJobMaxReturnTime = jobs[index - 1]?.maxReturnTime;

const initialDeadlineDate = useMemo(() => {
  if (job.returnTime) {
    return currentJobDeadline;
  }

  const projectionAnchor = previousJobMaxReturnTime
    ? parseUtcDateString(previousJobMaxReturnTime)
    : referenceNowUtc;

  const projectedUtc = getProjectedReturnTime(
    parseJobTiming(job),
    projectionAnchor
  );

  return getNewDateWithUserTimezonesDiff({
    date: projectedUtc,
    systemTimeZone: timeZone,
    reverse: true
  });
}, [job, currentJobDeadline, referenceNowUtc, timeZone, previousJobMaxReturnTime]);
```

### 3. `addNewJobs` schema requires `returnTime` while its TS type allows undefined

**[File: apps/creative-portal/api/mixtures/jobs/addNewJobs/schema.ts]**

**Function/Class:** `addNewJobsRequestSchema.body.jobs[].jobData.returnTime` (line 12)

**Severity:** medium

**Problem:** `returnTime: Yup.string().required()` enforces a required string, but `BulkJobData.jobData.returnTime` in `addNewJobs/types.ts` line 9 is typed `returnTime?: string`. Schema validation rejects undefined, while callers and TS believe undefined is legal.

**Impact:** First request that omits `returnTime` (e.g., creating an unassigned job in a dynamic-return-times workflow) hits a 400 from Yup before it reaches OMS. The PR description acknowledges this as a known follow-up but the code drift remains in scope of this PR. Recommend either making the schema match the type (`Yup.string().optional()`) or making the type match the schema and revisiting callers that omit it.

**Fix:** Drop `.required()` to match the type if no creation flow actually requires it today; otherwise tighten the type. Since the PR also widened `JobLookup.returnTime`, `Job.returnTime`, `JobFromLookup.returnTime` to optional, the schema is the outlier and should follow.

### 4. `jobItemToJobTableItem` — unnecessary optional chain and dead fallback

**[File: apps/creative-portal/components/pages/jobs/utils.ts]**

**Function/Class:** `jobItemToJobTableItem` (lines 30–35)

**Severity:** low

**Problem:**

```typescript
const effective = jobItem.maxReturnTime
  ? getEffectiveDeadline(parseJobTiming(jobItem))?.toISOString()
  : jobItem.returnTime;
```

`getEffectiveDeadline` returns `Date` (non-optional), so `?.toISOString()` is unreachable. The `: jobItem.returnTime` branch only fires when `maxReturnTime` is falsy, but `Job.maxReturnTime` is typed as required `string` — so this branch is dead unless OMS sends an empty string, which the parsing layer doesn't handle anyway.

**Impact:** Minor — code reads like it handles a case it can't reach. A future reader assumes `jobItem.returnTime` is a valid fallback shape; if `Job.maxReturnTime` is ever loosened to optional, this fallback would emit a Z-less OMS string into a slot where downstream code expects a `.toISOString()` Z-suffix, mixing shapes.

**Fix:**

```typescript
const effective = getEffectiveDeadline(parseJobTiming(jobItem)).toISOString();
```

`Job.maxReturnTime` is now required at the type level, so `parseJobTiming` will not throw for any well-typed input. If you want defense-in-depth, guard at the type-narrowing level.

### 5. `OrderStatusCardInfo/template.tsx` — infinite spinner for unassigned current job

**[File: apps/creative-portal/components/molecules/OrderStatusCardInfo/template.tsx]**

**Function/Class:** `DeadlineRow` (lines 26–56) and the User DD row (lines 177–182)

**Severity:** low

**Problem:** `DeadlineRow` renders an `IconLoading` spinner whenever `dateTime` is null/undefined and `showLoadingFallback` is true. The hook (`OrderStatusCardInfo/hooks.ts`) sets `returnTime = currentJobData?.returnTime` — undefined for any unassigned job. For unassigned current jobs in "In Queue" / "On Hold" the spinner appears next to "User DD:" indefinitely (the job-lookup query has resolved, but `returnTime` is genuinely absent).

**Impact:** Misleading UI — admin sees a spinner that suggests data is still loading when the value is permanently absent. The Jira ticket Requirement 5.2 calls for "a fallback indicator rather than displaying 'Invalid Date' or crashing" — a spinner technically isn't "Invalid Date," but it's not a sensible fallback either. Likely pre-existing behavior (the old template had inline duplicated rows with similar logic), but the PR re-architected this row and could fix it here.

**Fix:** Distinguish "still loading" from "no value":

```typescript
const DeadlineRow = ({
  dateTime,
  label,
  isLoading = false,
  showReturnTime
}: { ... }) => (
  <Styled.Item sx={{ flexWrap: "nowrap" }}>
    <Styled.Label>{label}</Styled.Label>
    <Styled.Value>
      {isLoading ? (
        <CreatingLoadingIconWrapper><IconLoading /></CreatingLoadingIconWrapper>
      ) : dateTime ? (
        <Flex sx={{ flexDirection: "column" }}>
          <DeadlineDisplay dateTime={dateTime} showReturnTime={showReturnTime} />
        </Flex>
      ) : (
        <span>—</span>
      )}
    </Styled.Value>
  </Styled.Item>
);
```

Then thread `isJobLoading` through and pass it to the User DD row only.

### 6. `runGuards` predicate evaluation runs eagerly across all guards

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturn.ts]**

**Function/Class:** `updateMaxReturnTime` and `updateReturnWindow` (`runGuards` consumers)

**Severity:** low

**Problem:** The guard arrays compute every `predicate` at call time, e.g.:

```typescript
runGuards([
  ...
  {
    errorText: "Job Due Date cannot be earlier than the locked User Due Date",
    predicate:
      job.returnTime !== undefined &&
      picked < parseUtcDateString(job.returnTime)
  }
]);
```

Each `parseUtcDateString` call happens regardless of whether the prior guard fired. Cheap in absolute terms but unnecessary; more importantly, it means a buggy predicate at index N can throw even when the guard at index 0 should have short-circuited.

**Impact:** Marginal — current predicates are pure and don't throw, so no functional issue. The pattern just denies you the lazy-evaluation safety net that `if (a) return X; if (b) return Y;` provides.

**Fix:** Convert `predicate: boolean` to `predicate: () => boolean` and call inside `runGuards`. Optional cleanup, defer if scope is tight.

### 7. Shift-mode cascade has no transactional rollback

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturn.ts]**

**Function/Class:** `updateMaxReturnTime` Shift branch (lines 245–273)

**Severity:** low

**Problem:** `Promise.all([mutatePartialJob(current), ...cascadeUpdates.map(mutatePartialJob)])` fires N+1 PUTs in parallel. If any one fails, the others have already mutated server state. There's no compensating action to revert the earlier successes.

**Impact:** Partial-update is recoverable (admin reopens the picker, the server values are reflected, they pick again). But the visible state is inconsistent during the failure window, and the toast (`showDefaultErrorToast` via `useJobMutation`) will fire for each failure independently — potentially N error toasts.

**Fix:** Either run sequentially with abort-on-first-error and surface a single rollup error, or move the cascade to a single OMS-side bulk endpoint (matches the deferred follow-up in the PR description: `OMS-side change` for `currentJobMaxReturnTime`). Acceptable to defer; flag for the follow-up ticket.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 113/113 files pass, 1,024/1,024 tests pass in `@proofed/creative-portal`; 103/103 files, 1,036/1,036 tests pass in `@proofed/shared` |
| `npx turbo run typecheck` | ✅ | 0 type errors across all workspaces (verified with `--force` to bypass cache) |
| `npx turbo run lint` | ⚠️ | 69 pre-existing prettier errors in `@proofed/shared` (e.g. `components/atoms/Fields/Select/hooks.ts`, `utils/sentryScrubber.test.ts`) and `@proofed/wysiwyg-editor` (`src/extensions/comments/index.ts`). Verified on `develop` — 72 errors exist there too, so this PR did not introduce them; it actually reduces the count by 3. **No new lint errors in any of the 64 files this PR touches.** |
| `npx turbo run build` | ✅ | `@proofed/creative-portal` and `@proofed/shared` build clean (95.83s + cached). All routes compile, no type warnings surfaced. |

---

## Tests

- ✅ `mergeJobPutBody.test.ts` — 7 tests including explicit-empty-string fallback semantics and incoming-id/orderId precedence
- ✅ `parseUtcDateString.test.ts` — 3 tests covering Z-suffix idempotence and fractional-seconds parsing
- ✅ `toHoursAndMinutes.test.ts` — extended with `-0` edge case (treated as positive zero) and `alwaysShowMinutes` precedence over `showEmptyHours`
- ✅ `formatTimeFromMinutes.test.ts` — extended with `alwaysShowMinutes` bypassing the days threshold, `-0` preservation
- ✅ `JobReturnTimesTray/index.test.tsx` — 12 tests, hook + component, deterministic frozen clock + Africa/Abidjan tz, mocked styles for jsdom
- ✅ `OrderJobs/utils.test.ts` — 327 lines covering `parseJobTiming`, `getProjectedReturnTime`, `getDisplayedReference`, `getDisplayedDurationMinutes`, `getAtRiskBufferMinutes`, `isAtRisk`, `isOverdue`, `isAssigned`, `updateJobReturnTimes`, `getDownstreamJobConstraint`
- ❌ **No test for `areDatesInSequence` × `updateJobReturnTimes` interaction** — would have caught Issue 1
- ❌ **No test for the picker/tray anchor consistency** (Issue 2) — both produce dates from input; a single property test on "for unassigned subsequent job, picker.initialDate.toISOString() === tray.leftDate.toISOString()" would lock the contract
- ⚠️ **Manual QA pending** per the PR's testing checklist: comment-55721 Scenario 2 walkthrough (cascade ceiling enforcement) and DevTools verification of the API surface

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Sequence guard regression (Issue 1); picker/tray anchor mismatch (Issue 2) |
| Regression risk | ⚠️ Medium — Issue 1 silently breaks an existing safety net; Issue 2 is visible only on multi-job orders |
| Tests | ⚠️ Strong coverage on new helpers, but gaps around the regression vectors |
| Code quality | ✅ Pure helpers, hook-driven UI, discriminated picker union, frame contract documented |
| Validation suite | ⚠️ test ✅, typecheck ✅, build ✅, lint ⚠️ pre-existing on `develop` (not introduced by this PR) |
| Mergeable state | ✅ Clean (no conflicts with `develop`) |

---

## Recommendation

**Request changes**

1. **Fix Issue 1 (high)** — replace `new Date(\`${value}Z\`)` with `parseUtcDateString(value)` in the `updatedJobsReturnTime` build, and add a unit test that feeds `updateJobReturnTimes` output back into `areDatesInSequence` for a known-bad sequence.
2. **Fix Issue 2 (medium)** — anchor `initialDeadlineDate` on `previousJob?.maxReturnTime` (parsed via `parseUtcDateString`) for unassigned subsequent jobs so the picker opens on the same date the tray shows.
3. **Decide on Issue 3 (medium)** — either drop `.required()` from `addNewJobs/schema.ts` or tighten the TS type to match the schema. Carrying the drift across this PR's wide surface area is risky.
4. **Defer Issues 4–7 (low)** — minor cleanups; can land in the same follow-up that extracts `useJobCard`.
5. **Walk through Scenario 2** from comment-55721 in a running app (the manual-QA checkbox is still unchecked in the PR description).
6. **Lint is not a blocker** — the failures live on `develop` and are unrelated to this PR's diff. Worth raising as a separate cleanup ticket.
