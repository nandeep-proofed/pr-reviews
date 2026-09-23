# PR Review: fix/PP-2201: Count Last Reviewed jobs from the editor's last review

**PR:** https://github.com/Proofed/B2BWebserver/pull/2497
**Jira:** https://proofed.atlassian.net/browse/PP-2201
**Commits reviewed:** cf37a88b0, edae837c7
**Diff:** 4 files, +351 / -24 (all in `apps/creative-portal/components/molecules/UserPreview/partials/EditorFeedback/`)
**Verdict:** No high-severity bugs. Three low-severity edge cases, none blocking.

> Note: this is a self-review of my own change. The separate verification pass was
> skipped on request, so the three findings below are unverified reads of the code,
> not reproduced failures.

---

## What this means for users (non-technical summary)

1. **The fix does what Adam asked for.** The "Last Reviewed: N jobs" hover now
   counts back to and including the last reviewed job, and leaves out the job
   being viewed. Adam's own example (Nandeep E-R-Re-QA on 29OctPartner) goes from
   "8 jobs" to "1 job", which is what he expected.
2. **Old unreviewed jobs no longer inflate the number.** Jobs from before the last
   review drop out. Example: ca we on order 21501 goes from 6 to 4.
3. **Reviews on canceled orders count.** The reviewed job is taken from the same
   data as the feedback dots, so a review on a canceled order still anchors the
   count.
4. **Three rare corner cases can show a slightly wrong line.** Each needs an
   unusual situation (a hover on an on-hold or canceled order, a failed request,
   or reviews written out of order). None affects the normal daily use.

---

## Findings

### 1. Reviewed job on a non-Live, non-Complete order counts as +1 (Low)

`utils.ts:42`

`isLastReviewedJobCurrent` is only true when the job search returned the reviewed
job on the current order. The search asks only for "Live" and "Complete" orders.
If the order being viewed has another status (on hold, canceled) and its job is
the last reviewed one, the flag is false and the reviewed job is added again.

- **Example:** reviewed job 101 sits on the viewed order, which is on hold. The
  hover reads "1 job" where it should read "0 jobs".
- **Fix, if wanted:** needs the reviewed job's order ID, which the assessment data
  does not carry. Not worth a change for this case.

### 2. Count and "days ago" can come from two different reviews (Low)

`index.tsx:192`

The count anchors on the highest reviewed `jobId`. `calculateDaysSinceLastFeedback`
uses `userFeedbacks.at(-1)`, the newest assessment by assessment id. They differ
only when an older job is reviewed after a newer one.

- **Example:** job 102 is reviewed, then job 100. The line reads "1 job / N days
  ago", where "1 job" counts from 102 but "N days" is the age of the review of 100.
- **Note:** the ticket says the days figure stays unchanged, and this is how it
  already worked. The test "dates the count from the newest reviewed job, not the
  newest feedback" pins the count side only.

### 3. A failed assessments request shows a confident never-reviewed count (Low)

`index.tsx:196`

If the job-assessments request fails, `useUserFeedbacks` falls back to `[]`. The
hover then shows "Not yet reviewed (N jobs)", stating the editor was never
reviewed and adding a count. Before this PR the same failure showed the same false
"Not yet reviewed", but without a number. The job-count path already shows "-"
when its data is missing, so this path is inconsistent with that rule.

- **Fix, if wanted:** expose the error state from `useUserFeedbacks` and show "-"
  when the assessments are unavailable. Touches a shared hook, so a small
  follow-up rather than part of this PR.

---

## What was checked and holds up

- **Core rule:** anchors on the highest reviewed `jobId`, adds 1 for the reviewed
  job unless it is on the current order, and leaves out every job on the current
  order. Matches the ticket comment and Adam's example.
- **Dropping the `!reviewScore` filter is safe:** no scored job can have an id
  above the highest reviewed `jobId`, so scored jobs never inflate the count.
- **The "-" on a failed job search** (PP-2100 QA Finding 3) is unchanged and still
  tested.
- **Tests:** 37 passing in `EditorFeedback` (every agreed ticket case, the stale
  job case, a reviewed job the search did not return, the never-reviewed count).
- **Security:** `/security` review found no issues (display logic only, no new
  requests, no unsafe rendering).

## Recommendation

Merge as is. Mention findings 1 and 2 as known edge cases. Consider finding 3 as a
small follow-up.
