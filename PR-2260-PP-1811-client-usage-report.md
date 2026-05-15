# PR Review: feature/PP-1811: Add Client Usage Report to Admin Portal reporting section

**PR:** https://github.com/Proofed/B2BWebserver/pull/2260
**Jira:** https://proofed.atlassian.net/browse/PP-1811
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Add Usage report entry to Admin Portal reporting section | New "Customer" tab added to `ReportingPage` at `/reporting#customer` | ✅ Addressed |
| Accessible to all authenticated admin users | `getServerSideProps` expanded to Admin, Superadmin, Returner, ServiceDelivery, ServiceSupport | ⚠️ Partial — see Issue #1 |
| Organization filter — single-select, populated from Org Search API | `OrganizationFilter` component using `useAllOrganizationsQuery({ status: "A" })` | ✅ Addressed |
| Must select org before report loads | Query disabled when `selectedOrgId` is null; prompt shown | ✅ Addressed |
| Project filter — multi-select org groups, default All | `ProjectFilter` (shared) with fallback to all org group IDs | ✅ Addressed |
| Changing org resets project filter to All | `handleOrgChange` calls `setSelectedOrgGroupIds([])` | ✅ Addressed |
| Data from OMS Charge API (complete orders only) | API handler filters `order.status === "Complete"` | ✅ Addressed |
| Report only requests data when org selected or filters change | React Query `enabled` gated on `selectedOrgId && activeOrgGroupIds.length > 0` | ✅ Addressed |
| Charge requests batched per org group | Creative portal fans out per orderId (not per group) due to API limitation; documented in code comment | ⚠️ Different approach — see Issue #2 |
| Single Complete section of summarized charge usage | `SummaryCards` with `showInProgress={false}` + `UsageChart` | ✅ Addressed |
| Sum charges same as Customer Portal | Uses shared `calculateChargeTotalFromCharges` | ✅ Addressed |
| No org selected → prompt | `OrganizationPrompt` rendered when `!selectedOrgId` | ✅ Addressed |
| No data → "No Usage In This Period" | `SummaryCards` shows empty state when `completeOrderCount === 0` | ✅ Addressed |
| **Date Range filter** | Deferred per Jira (not in admin portal) — DateRangeFilter extracted to shared but not wired in admin | ✅ Correct |
| **Customer filter** | Deferred per Jira | ✅ Correct |
| **In Progress section** | Deferred per Jira — `inProgressOrders` always `[]` in admin API | ✅ Correct |

**Beyond scope:** Shared component extraction (SummaryCards, UsageChart, DateRangeFilter, ProjectFilter), customer portal refactor to use shared, `mapWithConcurrency` utility, `calculateOrderTotal` move to shared, `computeDateRange`/`computeOrderMetrics` extraction, theme additions. These are reasonable enabling work for code reuse.

---

## Architecture Analysis

The PR takes a well-structured approach:

1. **Shared extraction:** Reusable UI components and utilities are moved from customer portal to `@proofed/shared`, with a new `ProjectGroup` interface abstracting away portal-specific org group shapes.
2. **Admin API:** A new `/api/mixtures/usageReportData` endpoint in the creative portal fetches orders per org group, then fans out charge fetches per orderId (bounded by `mapWithConcurrency` at concurrency 10). This differs from the customer portal's per-group batching because the Charge API's `SearchBy=organizationGroupId` response omits `orderId`.
3. **Component structure:** Follows project conventions — `index.tsx` + `hooks.ts` + `styles.ts` + `types.ts` per component folder.
4. **Customer portal regression:** Existing components are updated to import from shared; types moved to shared types. Tests updated accordingly.

The `mapWithConcurrency` utility is a clean, well-tested concurrency limiter that avoids pulling in a dependency like `p-limit`.

---

## Issues Found

### 1. Role expansion may be too permissive

**[File: apps/creative-portal/components/pages/reporting/index.tsx]**
**Function/Class:** `getServerSideProps`
**Severity:** medium
**Problem:** The `getServerSideProps` now includes `UserRole.Returner` and `UserRole.ServiceSupport` in addition to Admin, Superadmin, and ServiceDelivery. The Jira ticket says "accessible to all authenticated admin users" but doesn't explicitly define which roles qualify. Returner and ServiceSupport may not be considered "admin users."
**Impact:** Users with Returner or ServiceSupport roles could access client financial data (charge totals) they shouldn't see.
**Fix:** Confirm with the product team whether Returner and ServiceSupport should have access to the Client Usage Report. If not, revert to the original role set (Admin, Superadmin, ServiceDelivery).

### 2. Order fan-out could be expensive for large organizations

**[File: apps/creative-portal/api/mixtures/usageReportData/getUsageReportData.ts]**
**Function/Class:** `getUsageReportData`
**Severity:** medium
**Problem:** The creative portal fetches charges per orderId (not per org group like the customer portal) because the Charge API's `SearchBy=organizationGroupId` omits `orderId`. For organizations with hundreds of completed orders, this means hundreds of individual HTTP requests to the charge service, even with the concurrency cap at 10. There's no pagination or limit on the number of orders processed.
**Impact:** Slow response times and high load on the charge service for large organizations. The endpoint could time out for organizations with thousands of completed orders.
**Fix:** Consider:
1. Adding a comment documenting the expected order-count scale
2. Adding a cap/warning log when order count exceeds a threshold (e.g., 500)
3. The code comment already mentions a follow-up to extend the Charge API — ensure a ticket is created for this

### 3. Unbounded order fetching — all orders for org group including very old ones

**[File: apps/creative-portal/api/mixtures/usageReportData/getUsageReportData.ts]**
**Function/Class:** `getUsageReportData`
**Severity:** medium
**Problem:** `fetchOrders` with `includeClosed: true` retrieves ALL completed orders for each org group, regardless of age. The `MIN_REPORT_DATE_ISO` floor is only applied after fetching and processing charges. If an org has 5 years of history, all orders are fetched, all charges are fetched, and then most are discarded.
**Impact:** Wasted API calls and processing time. An organization with a long history will make far more charge requests than necessary.
**Fix:** If the `fetchOrders` API supports a date filter, use it to limit orders to those created after a reasonable lookback (e.g., 2 years before `MIN_REPORT_DATE_ISO`). If not, document this limitation as a follow-up.

### 4. Date string comparison for `MIN_REPORT_DATE_ISO` is fragile

**[File: apps/creative-portal/api/mixtures/usageReportData/getUsageReportData.ts]**
**Function/Class:** `getUsageReportData` (line ~120 in diff)
**Severity:** low
**Problem:** The code compares `earliestChargeTimestamp.split("T")[0] < MIN_REPORT_DATE_ISO` using string comparison. This works for ISO dates (lexicographic order matches chronological order), but is fragile — if the timestamp format ever changes, this silently breaks.
**Impact:** Low — ISO format is stable, but the pattern is non-obvious to future maintainers.
**Fix:** Consider using a Date comparison for clarity:

```typescript
const chargeDate = new Date(earliestChargeTimestamp);
const minDate = new Date(MIN_REPORT_DATE_ISO);
if (chargeDate < minDate) return null;
```

### 5. `useUsageReportDataQuery` enabled logic has redundant `|| false`

**[File: apps/creative-portal/services/usageReportData/index.ts]**
**Function/Class:** `useUsageReportDataQuery`
**Severity:** low
**Problem:** The `enabled` option is:
```typescript
enabled:
  (options?.enabled !== false &&
    !!params &&
    params.organizationGroupIds.length > 0) ||
  false
```
The `|| false` is redundant — the boolean expression already evaluates to `false` when the conditions aren't met.
**Impact:** No functional issue; just confusing to read.
**Fix:** Remove `|| false`:

```typescript
enabled:
  options?.enabled !== false &&
  !!params &&
  params.organizationGroupIds.length > 0
```

### 6. Inline styles in OrganizationFilter hooks

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/partials/OrganizationFilter/hooks.ts]**
**Function/Class:** `useOrganizationFilter`
**Severity:** low
**Problem:** The `styles` memo hardcodes CSS values (`borderRadius: "0.375rem"`, `minHeight: "2.25rem"`, `boxShadow: "0 1px 1px rgba(0, 0, 0, 0.05)"`) inline rather than using theme tokens. This is inconsistent with the project's convention of using theme values for styling.
**Impact:** Harder to maintain if the design system changes; doesn't follow the co-located styles pattern.
**Fix:** Move these values to the theme or use existing theme tokens. At minimum, use `theme.shadows` for the box-shadow.

### 7. `calculateOrderTotal` removes test for `processingFeeRate` derivation

**[File: packages/shared/utils/usageReport/calculateOrderTotal.test.ts]**
**Function/Class:** `calculateOrderTotal` test
**Severity:** low
**Problem:** The test "does not derive fee from rate fields" was removed during the move from customer portal to shared. This test verified that `processingFeeRate`/`processingFeeQuantity`/`processingFeeUnit` are NOT used to derive the processing fee — an important behavioral guarantee.
**Impact:** If someone later adds fee derivation from rate fields, there's no test guarding against it.
**Fix:** Consider keeping the test, adapted to the new `OrderDetailForCalculation` type. If the type no longer has those fields, the test may have been correctly removed — but document this decision.

### 8. `year` field added to `MonthlyUsageData` but customer portal test mock could be more explicit

**[File: apps/customer-portal/components/pages/reports/usage/index.test.tsx]**
**Function/Class:** test mock data
**Severity:** low
**Problem:** The `year` field was added to `MonthlyUsageData` and the test mock data was updated. However, the mock has `year: 2026` for all entries. While this is fine for testing, the customer portal's `hooks.ts` now adds `year` from the month key parsing — this path should be explicitly tested.
**Impact:** If the year parsing in customer portal hooks breaks, existing tests wouldn't catch it since they mock the entire hook return.
**Fix:** Add a test case in `hooks.test.ts` that verifies the `year` field is correctly derived from order dates spanning multiple years.

### 9. `formatUsageCost` not exported from shared barrel

**[File: packages/shared/utils/usageReport/index.ts]**
**Function/Class:** barrel export
**Severity:** low
**Problem:** `formatUsageCost` is exported from `packages/shared/utils/usageReport/index.ts`, which is correct. However, `SummaryCards` and `CustomTooltip` import it directly from `@proofed/shared/utils/usageReport` (the barrel). This is fine but worth verifying the barrel export includes it — confirmed it does via `export { formatUsageCost } from "./formatUsageCost"`.
**Impact:** None — this is working correctly.
**Fix:** No fix needed.

### 10. `isClientApi: false` hardcoded in creative portal charge processing

**[File: apps/creative-portal/api/utils/charges/fetchChargesForOrder.ts]**
**Function/Class:** `fetchChargesForOrder`
**Severity:** low
**Problem:** `enhanceTotalChargeEntry` is called with `isClientApi: false`. This is correct for the admin/creative portal context. However, the customer portal's `fetchChargesForOrder` presumably uses `isClientApi: true`. The two implementations have diverged — the creative portal version adds `isBeforeChargeData` logic and minimum charge fallback that may not exist in the customer portal version.
**Impact:** The creative portal's charge processing is more sophisticated than the customer portal's, which could lead to charge total discrepancies between the two portals for the same orders.
**Fix:** Document the intentional divergence. If the charge processing should be identical, consider sharing the `fetchChargesForOrder` logic.

### 11. Color inconsistency between SummaryCards and CustomTooltip

**[File: packages/shared/components/molecules/UsageReport/UsageChart/partials/CustomTooltip/index.tsx]**
**Function/Class:** `CustomTooltip`
**Severity:** low
**Problem:** CustomTooltip uses `theme.colors.pastelGreen1` for the "Complete" dot, while SummaryCards uses `theme.colors.green1` for the complete accent color. The chart bars also use `green1`. This creates a visual inconsistency where the tooltip dot color doesn't match the bar color or the summary card accent.
**Impact:** Minor visual inconsistency in the report. Users may notice the tooltip dot color differs from the bar and card accent.
**Fix:** Use `theme.colors.green1` in the tooltip for consistency, or if `pastelGreen1` is intentional for tooltip legibility, add a comment explaining the design choice.

### 12. Props destructuring not in alphabetical order

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx]**
**Function/Class:** `ClientUsage`
**Severity:** low
**Problem:** The hook return destructuring in `ClientUsage/index.tsx` is not alphabetized:
```typescript
const {
  summary, monthlyData, orgGroups, selectedOrgId,
  handleOrgChange, selectedOrgGroupIds, ...
} = useClientUsageReport();
```
Per CLAUDE.md: "Destructuring in `index.tsx` follows the same order" (alphabetical).
**Impact:** Violates project convention. Same issue exists in `OrganizationFilter/index.tsx` and `OrganizationFilter/hooks.ts` where props are destructured as `selectedOrgId, onOrgChange, isLoading` instead of alphabetical `isLoading, onOrgChange, selectedOrgId`.
**Fix:** Reorder destructuring to alphabetical in all three files.

### 13. Missing unit test for OrganizationFilter hooks

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/partials/OrganizationFilter/]**
**Function/Class:** `useOrganizationFilter`
**Severity:** low
**Problem:** `OrganizationFilter/hooks.ts` contains non-trivial logic (style computation, option mapping from organizations, change handler with type narrowing) but has no dedicated `hooks.test.ts`. The component test (`index.test.tsx`) tests integration but not the hook logic directly.
**Impact:** Hook-level logic changes could regress without detection.
**Fix:** Add `OrganizationFilter/hooks.test.ts` to directly test option building, style customization, and the change handler callback.

---

## Tests

- ✅ `getUsageReportData` (creative-portal): 359 lines, covers org group deduplication, order status filtering, charge timestamp bucketing, MIN_REPORT_DATE filtering, no-charge orders, failed charge fetches, concurrency cap — comprehensive
- ✅ `fetchChargesForOrder`: 252 lines, covers isBeforeChargeData paths, minimumChargeAmount fallback, synthesized total charge, error resilience, logger integration — thorough
- ✅ `useClientUsageReport` hooks: 171 lines, covers org selection, project reset, fallback to all groups, loading state, summary/monthly derivation — good
- ✅ `OrganizationFilter`: 181 lines, covers rendering, org query, selection, clearing, loading state — good
- ✅ `SummaryCards` (shared): 78 lines, covers rendering, showInProgress toggle, empty state — adequate
- ✅ `ProjectFilter` (shared): updated from customer portal, adapted to `ProjectGroup` type — good
- ✅ `CustomTooltip` (shared): 117 lines, covers active/inactive, year display, % complete calculation — good
- ✅ `UsageChart/utils`: 147 lines, covers formatCostTick edge cases, calculateNiceTicks with integerOnly — thorough
- ✅ `mapWithConcurrency`: 75 lines, covers ordering, concurrency limit, empty input, error propagation — solid
- ✅ `computeOrderMetrics`: 62 lines, covers empty, aggregation, missing dates — good
- ✅ `computeDateRange`: 86 lines, covers all options, min date clamping, custom dates — good
- ✅ `calculateOrderTotal`: moved from customer portal, adapted types, removed some comments — adequate
- ✅ Customer portal `getUsageReportData` test: 207 lines — new test added for existing endpoint
- ⚠️ No test for `ClientUsage/index.tsx` component rendering (only hooks are tested) — the component's conditional rendering logic (error states, loading skeleton, no-org prompt) is not directly tested
- ⚠️ No test for `OrganizationFilter/hooks.ts` — hook logic (style computation, option mapping, change handler) not unit tested
- ⚠️ No integration test for the full filter cascade flow (org → project → data refresh)
- ⚠️ `computeOrderMetrics` lacks tests for malformed/null `creationDateTime` and invalid ISO strings
- ⚠️ `useClientUsageReport` hooks test lacks error state scenarios (query errors, null data)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Implementation matches Jira requirements (reduced scope) |
| Regression risk | ⚠️ Medium — customer portal refactored to shared imports; charge processing differs between portals |
| Tests | ✅ Comprehensive — 1,884 tests passing per PR description |
| Code quality | ✅ Clean architecture, good separation, follows project conventions |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. **Verify role expansion** (Issue #1): Confirm with product that Returner and ServiceSupport should access the Client Usage Report. This is a security/authorization concern.
2. **Document scalability limitations** (Issues #2, #3): The per-orderId charge fan-out and unbounded order fetch could be problematic for large organizations. Add logging/metrics and create a follow-up ticket to optimize.
3. **Remove redundant `|| false`** (Issue #5): Minor cleanup.
4. **Alphabetize destructuring** (Issue #12): Fix in `ClientUsage/index.tsx`, `OrganizationFilter/index.tsx`, and `OrganizationFilter/hooks.ts`.
5. **Consider adding a `ClientUsage` component test** for the rendering paths (error banner, loading state, org prompt).
6. **Consider keeping the removed `processingFeeRate` test** (Issue #7) or documenting why it was removed.
7. **Add error state tests** for `useClientUsageReport` hook (query failures, null data scenarios).

Overall, this is a well-structured PR with good test coverage. The shared component extraction is clean and the admin-specific additions follow established patterns. The main concerns are around authorization (role expansion) and scalability (charge fan-out for large orgs).
