# PR Review: fix/PP-2201: Count Last Reviewed jobs from the editor's last review

**PR:** https://github.com/Proofed/B2BWebserver/pull/2497
**Jira:** https://proofed.atlassian.net/browse/PP-2201
**Status:** In Progress (Bug, Highest)
**Commits reviewed:** cf37a88b0, edae837c7

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
2. **The ticket description still describes an older rule.** The code follows
   what Adam agreed in his Jira comment and in Slack, not the description. Until
   the description is updated, QA may test the wrong numbers.
3. **If the review history fails to load, the hover can claim an editor was never
   reviewed and show a job count with it.** The "never reviewed" part already
   happened before this change; the count now makes it look trustworthy. It only
   happens when that request fails.
4. **Storybook shows a different figure from its description.** Developer-facing
   only; users are not affected.

---

## Requirement history

The rule changed several times. The PR follows the latest agreed version, which
is not the one in the ticket description.

| When | Source | What it says |
|---|---|---|
| PP-2100 (released) | QA Finding 3, Rafał | A failed job search shows "-", never a plausible wrong number |
| PP-2100 (released) | Adam, 18 Sep | Canceled orders do not count ("the numbers need to match") |
| PP-2201 | Ticket description | Leave the reviewed job out, count the hovered job, count canceled orders |
| PP-2201 | Adam, comment 76584 | The opposite convention: "don't count the current job; count back to and including the reviewed job" |
| PP-2201 | Slack, after commit cf37a88b0 | Adam rejected "0 jobs" where the newest job was reviewed (his example should read "1 job"). Canceled orders count only **where feedback was provided** (as in the feedback dots) |
| PP-2201 | Commit edae837c7 + comment 76601 | Implements the Slack rule. Adam has not replied on Jira yet |

The description and comment 76584 give the same number in the ticket's agreed
cases (1, 2 and 6 jobs). They differ when the job being viewed is not newer than
the last review, which is exactly Adam's example.

---

## Jira Requirements vs Implementation

| Requirement (source) | PR Implementation | Status |
|---|---|---|
| Count dated from the last review, not all unscored jobs (description) | Anchors on the highest reviewed `jobId` from the feedback | ✅ Addressed |
| Counting rule (Adam, 76584 + Slack) | Counts the reviewed job plus later jobs, leaves out the current order's job | ⚠️ Matches Adam, **contradicts the description**, which still says to count the hovered job and leave the reviewed job out. Description needs updating before QA |
| Cases 1 to 5: Not yet reviewed, (X jobs), 1, 2 and 6 jobs (description) | Covered by tests; same result under both rules | ✅ Addressed |
| Case 6: days since the last completed review (description) | `calculateDaysSinceLastFeedback` uses the newest review written, unchanged | ✅ Addressed |
| Singular "1 job" (description) | `formatJobCount` | ✅ Addressed |
| Canceled orders (description: all; Slack: only where feedback was provided) | The reviewed job comes from the feedback, so reviews on canceled orders count. Canceled-order jobs without feedback cannot be fetched (OMS Job Search section 25.1 accepts only Live or Completed) | ⚠️ Meets the Slack rule; **agreed in Slack only**, not written in Jira |
| Never reviewed: "(X jobs)" (description) | X leaves out the current job, following Adam's rule | ⚠️ Consistent with Adam's rule but **not written anywhere in Jira** |
| Never-reviewed design (Figma red "Never reviewed" or ticket text) | Ticket text used | ⚠️ Still Adam's open decision (76584) |
| Profile order history must match if canceled orders count (description notes) | Not touched | ⚠️ Only relevant if canceled jobs without feedback are ever counted |

No scope creep: all changes are inside `EditorFeedback`.

---

## Architecture Analysis

The counting rule moves out of the JSX into a pure helper,
`countJobsSinceLastReview(jobs, userFeedbacks, currentOrderId)` in `utils.ts`.
The component keeps its existing data sources (Live and Complete job searches by
user, plus job assessments), drops the `!reviewScore` filter so the whole history
is available, and passes the viewed `orderId`. Taking the reviewed job from the
assessments is what lets reviews on canceled orders count without a backend
change. The QA-approved "-" fallback for a failed job search is unchanged. The
helper and `formatJobCount` are used only by this component, and the
component's props are unchanged, so there is no cross-component regression
surface.

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

**Impact:** a misleading figure on a surface used to decide who to review next. QA already rejected this pattern for the job searches on PP-2100 (Finding 3: "showing a plausible wrong number on failure"), which is why a failed job search shows "-". The review-history request is the other half of the same line and has no such guard.

**Fix:** expose the error state from `useUserFeedbacks` and treat a failed load like a failed job search:

```ts
const { data: userFeedbacks = [], isLoading, isError } = useJobAssessments(...);
return { userFeedbacks, feedbackItems, isLoadingUserFeedbacks: isLoading, isUserFeedbacksError: isError };
```

Then set `jobCount` to `"-"` when the feedback failed with nothing cached, and add a test for it. The hook is shared, so this can be a small follow-up.

**Resolution:** Skipped. The failure case already showed a false "Not yet reviewed" before this PR, so the wrong state is not new; only the count is. The fix needs a change to the shared `useUserFeedbacks` hook, which the team score cell (`TableWithFilters/partials/TeamScoreCellContent`) also uses, so it reaches beyond PP-2201's scope. A failed request still raises the global error toast, so the failure is visible. To be raised as a follow-up ticket.

### 2. Storybook "Default" description no longer matches the rendered count

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

**Resolution:** Skipped. Developer documentation only, with no effect on users or on QA of the ticket, and the story file is outside the files this ticket changes. Can be picked up with the next Storybook update.

---

## Open Questions

- **Should "N jobs" and "days ago" refer to the same review?** The count starts from the highest reviewed job ID; the days figure is the age of the newest review written, which is what case 6 in the ticket asks for. They differ only when an older job is reviewed after a newer one (the test "dates the count from the newest reviewed job, not the newest feedback" builds this case). Not a defect against the ticket; a question for Adam. `utils.ts:9`
  - **Resolution:** Skipped. Case 6 in the ticket defines "days ago" as the time since the last completed review, and the code follows it. Reviews written out of job order are rare.
- Can the hover be opened on an order that is neither Live nor Complete (canceled, on hold)? If so, when the editor's job on that order is also their last reviewed job, the job search cannot return it and the hover would show "1 job" instead of "0 jobs". Fixing it would need the order's own job list. `utils.ts:43`
  - **Resolution:** Skipped. The orders table's status filter can show canceled orders, so the case may be reachable, but it needs the viewed order's job to also be the editor's last reviewed job, which is rare. The result is off by one (1 instead of 0), and a fix would need the order's own job list as an extra data source.
- A workflow can give the same editor two jobs of one type on one order. All of them are left out as "current". Is that the intended reading of "the current job"? `utils.ts:34`
  - **Resolution:** Skipped. All of the editor's work on the order being viewed is current work, so leaving it all out matches "leave out the current job". No change unless Adam says otherwise.
- For Review (QA) jobs, is `JobAssessment.jobId` the editor's own review job ID (comparable with the search results)? The BFF passes it through, so this depends on the OMS contract.
  - **Resolution:** Skipped. The feedback dots already rely on the same `jobId` and job type filter, so the count adds no new assumption. To be confirmed during QA on a reviewer with QA feedback.

---

## Resolution Summary

| # | Point | Severity | Resolution | Reason |
|---|---|---|---|---|
| Issue 1 | Failed review-history request shows a "never reviewed" count | Medium | Skipped | Pre-existing false state; fix needs the shared `useUserFeedbacks` hook (also used by the team score cell); follow-up ticket |
| Issue 2 | Storybook description out of date | Low | Skipped | Developer docs only; outside the ticket's files |
| Q1 | "N jobs" and "days ago" from different reviews | Question | Skipped | Days follow ticket case 6; rare |
| Q2 | Viewed order neither Live nor Complete | Question | Skipped | Rare; off by one at most; fix needs extra data |
| Q3 | Several jobs on the viewed order | Question | Skipped | All work on the viewed order is current work |
| Q4 | `JobAssessment.jobId` for QA jobs | Question | Skipped | Same assumption as the feedback dots; confirm in QA |

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | Skipped | User opted out. Affected tests run during development: 37/37 pass in `EditorFeedback` |
| `npx turbo run typecheck` | Skipped | User opted out. Passed for `@proofed/creative-portal` on commit edae837c7 during development |
| `npx turbo run lint` | Skipped | User opted out. ESLint and Prettier clean on the changed folder |
| `npx turbo run build` | Skipped | User opted out. Last build ran clean on commit cf37a88b0 only; not re-run on edae837c7 |

---

## Tests

- ✅ Unit tests for the new helper (`utils.test.ts`) and the component (`index.test.tsx`), 37 passing.
- ✅ Every agreed ticket case covered: 1, 2, 6 jobs; newest job reviewed and viewed from an older job (Adam's example); viewing the reviewed job itself; stale unscored jobs; reviewed job missing from the search (canceled order); never-reviewed count and singular.
- ✅ The existing "-" behaviour for a failed job search is still tested.
- ❌ No test for a failed review-history request (Issue 1).
- ⚠️ No test where the reviewed job is on the current order and newer jobs exist on other orders (logic reads correctly).

### Suggested manual QA script

Test against Adam's rule (comment 76584 + Slack), not the current description. On b2btest after merge, hover the editor name in the orders table and in the order side panel:

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
| Correctness | ✅ (against Adam's agreed rule) |
| Requirements traceability | ⚠️ Jira description out of date |
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

**Approve with suggestions.** The code is right; the paperwork is not.

Before merge:

1. **Adam updates the Jira description** to the Slack rule, or confirms it on Jira: count back to and including the reviewed job, leave out the current job; canceled orders count only where feedback was provided; never-reviewed "(X jobs)" leaves out the current job. This is the main blocker for QA.
2. Add a reviewer (none assigned yet) and re-run the build on edae837c7.
3. Adam decides the never-reviewed design (Figma red "Never reviewed" or the ticket text).

All review points are skipped with reasons (see Resolution Summary). Raise a
follow-up ticket for Issue 1.
