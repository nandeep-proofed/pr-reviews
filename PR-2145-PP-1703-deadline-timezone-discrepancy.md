# PR Review: fix/PP-1703: Deadline field shows 2-hour discrepancy between Workflow step and Order detail

**PR:** https://github.com/Proofed/B2BWebserver/pull/2145
**Jira:** https://proofed.atlassian.net/browse/PP-1703
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Deadline on Workflow step matches Order detail (no TZ shift) | Added `timeZone` parameter to `getFormattedDates()`, uses `formatInTimeZone()` + `utcToZonedTime()` when timezone is provided. Threads `user?.timeZone` from `WorkflowStep` → `DeadlineSection` → `getFormattedDates()` | ✅ Addressed |
| No regression in overdue or other deadline-based behaviour | `formatDateRange` migrated from `formatTz` to `formatInTimeZone` — behavioral change that also fixes timezone handling for non-TZ tokens | ⚠️ Needs manual verification |

---

## Architecture Analysis

The fix correctly addresses the root cause. The Workflow step was using `format()` from `date-fns` (browser local time), while the Order detail uses timezone-aware formatting. By passing `user?.timeZone` and using `formatInTimeZone`, both should now show the same timezone-adjusted time.

---

## Issues Found

### 1. Pre-existing bug in `formatDateRange` comparisons

```
[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/utils.tsx]
Function: formatDateRange
Severity: medium
Problem: formatDateRange (lines 165-195 in the PR's version)
still uses isSameMinute(minDate, maxDate), isSameDay(minDate,
maxDate), and minDate.getMonth() on raw UTC dates — not
timezone-adjusted. This means if two dates are on different
days in UTC but the same day in the user's timezone (or vice
versa), the range formatting could be wrong.
Impact: Incorrect date range display when dates cross day
boundaries differently in UTC vs user timezone.
Fix: Apply utcToZonedTime before these comparisons in
formatDateRange, same pattern as the new getFormattedDates
timezone block.
```

### 2. No tests added

```
[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/utils.tsx]
Function: getFormattedDates, formatDateRange
Severity: medium
Problem: The PR has no unit tests. getFormattedDates and
formatDateRange are pure utility functions — they should
have test coverage for the timezone path. The project
requires tests for all new code per CLAUDE.md.
Impact: No automated verification that timezone formatting
is correct. Regressions could slip through.
Fix: Add unit tests for getFormattedDates with timezone
parameter and for formatDateRange with the formatInTimeZone
migration.
```

### 3. `timeZone` fallback behavior

```
[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/components/DeadlineSection.tsx]
Function: DeadlineSection
Severity: low
Problem: If user?.timeZone is undefined, getFormattedDates
falls back to the old browser-local-time path. This means
the bug persists for users without a timezone set.
Impact: Users without a timezone configured will still see
the 2-hour discrepancy.
Fix: Confirm whether user.timeZone is always populated. If
not, consider a sensible default (e.g., "UTC" or
Intl.DateTimeFormat().resolvedOptions().timeZone).
```

### 4. `formatTz` → `formatInTimeZone` behavioral change in `formatDateRange`

```
[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/utils.tsx]
Function: formatDateRange
Severity: low
Problem: formatTz only applies timezone to TZ-specific tokens
(z, O, etc.), while formatInTimeZone converts the date to the
target timezone before formatting ALL tokens. This is a
behavioral change that broadens scope beyond getFormattedDates.
Impact: Could change what users see in the workflow window
input dates. This is likely also a bugfix, but it's an
undocumented side effect.
Fix: No code change needed, but verify manually that
formatDateRange callers (WorkflowWindowInput/hooks.ts lines
280, 670) display correctly after this change.
```

---

## Tests

- ❌ No unit tests for `getFormattedDates` with timezone
- ❌ No unit tests for `formatDateRange` with `formatInTimeZone` migration
- ⬜ Manual: Workflow step deadline matches Order detail side panel
- ⬜ Manual: Overdue and deadline-based behaviour not regressed
- ⬜ Manual: Workflow window input dates display correctly

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes core TZ discrepancy |
| Regression risk | ⚠️ Medium — `formatDateRange` comparisons still use raw dates, `formatTz` → `formatInTimeZone` broadens scope |
| Tests | ❌ Missing |
| Code quality | ✅ Clean, well-structured |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes**
1. Add tests for `getFormattedDates` with timezone parameter
2. Fix `formatDateRange` date comparisons to use zoned times (same pattern as the new `getFormattedDates` timezone block)
3. Fill in the PR description
