# PR Review: fix/PP-2201: Count Last Reviewed jobs from the editor's last review

**PR:** https://github.com/Proofed/B2BWebserver/pull/2497
**Jira:** https://proofed.atlassian.net/browse/PP-2201
**Status:** In Progress (Bug, Highest)

> Self-review of my own change. Lenses run: correctness/logic, regressions,
> error handling, React/performance, type safety, test quality, reuse and
> conventions. Accessibility and security lenses skipped (text-only change; a
> separate `/security` review already found nothing). Validation suite skipped at
> the user's request.

---

## What this means for users (non-technical summary)

1. **The count now matches what Adam asked for.** Adam's own example
   (Nandeep E-R-Re-QA on 29OctPartner) goes from "8 jobs" to "1 job", and old
   unreviewed jobs no longer inflate the number (ca we on 21501: 6 to 4).
2. **If the review history fails to load, the hover can claim an editor was never
   reviewed and show a job count with it.** The "never reviewed" part already
   happened before this change; the count now makes it look trustworthy. It only
   happens when that request fails.
3. **In a rare case the two halves of the line can describe different reviews.**
   If someone reviews an older job after a newer one, "N jobs" counts from the
   newer job while "days ago" is the age of the latest review written.
4. **Storybook shows a different figure from its description.** Developer-facing
   only; users are not affected.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Count dated from the last review, not all unscored jobs | `countJobsSinceLastReview` anchors on the highest reviewed `jobId` from the feedback | ✅ Addressed |
| Counting convention (Adam, comment 76584 and Slack): include the reviewed job, leave out the current job | Adds 1 for the reviewed job, excludes jobs on the viewed order | ✅ Addressed |
| Case 1: current job only, never reviewed → "Not yet reviewed" | No suffix when there is no other job | ✅ Addressed |
| Case 2: never reviewed, several jobs → "Not yet reviewed (X job/s)" | Suffix with X excluding the current job | ✅ Addressed |
| Cases 3 to 5: 1, 2 and 6 jobs | Covered by tests | ✅ Addressed |
| "M days ago" unchanged | `calculateDaysSinceLastFeedback` untouched | ✅ Addressed (see Issue 2) |
| Singular "1 job" | `formatJobCount` | ✅ Addressed |
| Canceled orders where feedback was provided count (Adam) | Reviewed job taken from feedback, which includes canceled orders | ✅ Addressed |
| Canceled orders in general (ticket description) | Not possible: OMS Job Search accepts only Live or Completed | ⚠️ Partial (Adam narrowed it to orders with feedback) |
| Never-reviewed Figma treatment | Ticket text used, not the red Figma state | ⚠️ Partial (open decision on the ticket) |

No scope creep: all changes are inside `EditorFeedback`.

---

## Architecture Analysis

The counting rule moves out of the JSX into a pure helper,
`countJobsSinceLastReview(jobs, userFeedbacks, currentOrderId)` in `utils.ts`.
The component keeps its existing data sources (Live and Complete job searches by
user, plus job assessments), drops the `!reviewScore` filter so the whole history
is available, and passes the viewed `orderId`. The QA-approved "-" fallback for a
failed job search is unchanged. The helper and `formatJobCount` are used only by
this component, and the component's props are unchanged, so there is no
cross-component regression surface.

---

## Issues Found

### 1. A failed review-history request shows a confident "never reviewed" count

**[File: apps/creative-portal/components/molecules/UserPreview/partials/EditorFeedback/index.tsx]**

> **In plain terms:** If the system cannot load an editor's review history, the hover says they were never reviewed and adds a job count, for example "Not yet reviewed (37 jobs)". A lead could wrongly treat a regularly reviewed editor as never reviewed, and the number makes the mistake look reliable.

**Function/Class:** EditorFeedback

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Log in to the admin area and open the orders page.
2. In DevTools, block or fail the request to `/api/jobAssessment` for an editor who has reviews.
3. Hover that editor's name.
4. **Expected:** no count presented as fact (for example "- jobs", matching how a failed job search is shown).
5. **Actual:** "Not yet reviewed (N jobs)", where N is every other job the editor has in that project.

**Problem:** `useUserFeedbacks` turns a failed request into an empty list, which the component cannot tell apart from "never reviewed". This PR adds a count to that state.

**Evidence:** `hooks/useUserFeedbacks.ts:20` `data: userFeedbacks = [],` with no error flag exposed. `index.tsx:197-200`:

```tsx
{`Not yet ${isReviewJob ? "QA'd" : "reviewed"}`}
{jobCount !== "-" &&
  jobCount > 0 &&
  ` (${formatJobCount(jobCount)})`}
```

With empty feedback, `utils.ts` returns `otherJobs.length`, so any editor with other jobs gets a suffix.

**Impact:** a misleading figure on a surface used to decide who to review next. Before this PR the same failure showed a bare "Not yet reviewed".

**Fix:** expose the error state from `useUserFeedbacks` and treat a failed load like a failed job search:

```ts
const { data: userFeedbacks = [], isLoading, isError } = useJobAssessments(...);
return { userFeedbacks, feedbackItems, isLoadingUserFeedbacks: isLoading, isUserFeedbacksError: isError };
```

Then set `jobCount` to `"-"` when the feedback failed with nothing cached, and add a test for it. The hook is shared, so this can be a small follow-up.

### 2. "N jobs" and "days ago" can come from different reviews

**[File: apps/creative-portal/components/molecules/UserPreview/partials/EditorFeedback/index.tsx]**

> **In plain terms:** If a reviewer scores an older job after a newer one, the hover counts jobs from the newer review but shows the age of the later-written one. The line still looks consistent, so nobody would notice, but the two numbers describe different reviews.

**Function/Class:** EditorFeedback, countJobsSinceLastReview, calculateDaysSinceLastFeedback

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Pick an editor with two jobs in one project, A (older) and B (newer).
2. Review B, then a few days later review A.
3. Hover the editor on a newer job.
4. **Expected:** both numbers refer to the same review.
5. **Actual:** "N jobs" counts from B, "days ago" is the age of the review of A.

**Problem:** the count anchors on the highest reviewed job ID, while the days use the newest assessment by assessment ID.

**Evidence:** `utils.ts` `Math.max(...userFeedbacks.map((feedback) => feedback.jobId))` versus `utils.ts:9` `const lastReviewedFeedback = userFeedbacks.at(-1);`, where `hooks/useUserFeedbacks.ts:34` sorts by assessment `id`. The test "dates the count from the newest reviewed job, not the newest feedback" builds exactly this case (`reviewedOn(102, 100)`) and checks only the count.

**Impact:** rare and small; reviews are almost always written in job order. The ticket says the days figure stays unchanged, which is why it was left alone.

**Fix:** product call. If both should mean "the last reviewed job", date the days from the assessment whose `jobId` is the maximum. Otherwise leave as is.

### 3. Storybook "Default" description no longer matches the rendered count

**[File: apps/creative-portal/components/molecules/UserPreview/index.stories.tsx]**

> **In plain terms:** The component library page for the user hover says it shows "3 jobs", but it now shows "1 job". Only developers and designers see this; users are not affected.

**Function/Class:** Default story

**Severity:** low

**Confidence:** high

**How to spot it:** code health, not user-reproducible. Open Storybook, "Molecules/User preview", Default story, and compare the footer with the story description.

**Problem:** the seeded jobs 301, 302 and 303 all sit on the current order, so the new rule leaves them all out and counts only the reviewed job.

**Evidence:** `index.stories.tsx:117,124,131` `orderId: ORDER_ID` for jobs 301 to 303; line 251 describes "'Last Reviewed: 3 jobs / <n> days ago'". New count: reviewed max 102 is not on the current order, no other jobs, so 0 + 1 = "1 job".

**Impact:** stale docs; breaks the project rule that stories mirror production.

**Fix:** give jobs 301 and 302 their own order IDs and keep 303 on `ORDER_ID`, which renders "3 jobs" (reviewed job 102 plus 301 and 302) and matches the existing description.

---

## Open Questions

- Can the hover be opened on an order that is neither Live nor Complete (canceled, on hold)? If so, when the editor's job on that order is also their last reviewed job, the job search cannot return it and the hover would show "1 job" instead of "0 jobs". Fixing it would need the order's own job list. `utils.ts:43`
- A workflow can give the same editor two jobs of one type on one order. All of them are left out as "current". Is that the intended reading of "the current job"? `utils.ts:34`
- For Review (QA) jobs, is `JobAssessment.jobId` the editor's own review job ID (comparable with the search results)? The BFF passes it through, so this depends on the OMS contract.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | Skipped | User opted out. Affected tests run during development: 37/37 pass in `EditorFeedback` |
| `npx turbo run typecheck` | Skipped | User opted out. Passed for `@proofed/creative-portal` on commit edae837c7 during development |
| `npx turbo run lint` | Skipped | User opted out. ESLint and Prettier clean on the changed folder |
| `npx turbo run build` | Skipped | User opted out. Last build ran clean on commit cf37a88b0 only |

---

## Tests

- ✅ Unit tests for the new helper (`utils.test.ts`) and the component (`index.test.tsx`), 37 passing.
- ✅ Every agreed ticket case covered: 1, 2, 6 jobs; newest job reviewed and viewed from an older job; viewing the reviewed job itself; stale unscored jobs; reviewed job missing from the search (canceled order); never-reviewed count and singular.
- ✅ The existing "-" behaviour for a failed job search is still tested.
- ❌ No test for a failed review-history request (Issue 1).
- ❌ Issue 2's test checks the count but not the days figure.
- ⚠️ No test where the reviewed job is on the current order and newer jobs exist on other orders (logic reads correctly).

### Suggested manual QA script

On b2btest after merge, hover the editor name in the orders table and in the order side panel:

1. Nandeep E-R-Re-QA on any live 29OctPartner order: "1 job / 15 days ago" (Adam's example).
2. ca we on 21501: "4 jobs / 203 days ago".
3. Gaurav M on 21523: "1 job / 13 days ago".
4. Nandeep B on 21576: "1 job / 49 days ago" (singular).
5. Gaurav M as reviewer on 21436: "Not yet QA'd (10 jobs)".
6. Calum S as reviewer on 21501: "Not yet QA'd" with no count.
7. Issue 1: block `/api/jobAssessment` in DevTools and hover a reviewed editor; note what shows.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ⚠️ (Issue 1 path untested) |
| Accessibility | n/a |
| Error handling | ⚠️ (Issue 1) |
| Security | ✅ (`/security` run, no findings) |
| Code quality | ✅ |
| Validation suite | Skipped (user opted out) |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Approve with suggestions**

1. Update the Storybook seed data so the Default story matches its description (Issue 3); a two-line change.
2. Decide whether to handle a failed review-history request in this PR or a follow-up (Issue 1).
3. Leave Issue 2 unless Adam wants both numbers tied to the same review.
4. Validation suite was not run as part of this review; re-run test, typecheck, lint and build before merging.
