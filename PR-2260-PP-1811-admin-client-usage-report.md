# PR Review: feature/PP-1811: Add Client Usage Report to Admin Portal reporting section

**PR:** https://github.com/Proofed/B2BWebserver/pull/2260
**Jira:** https://proofed.atlassian.net/browse/PP-1811
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Add "Usage" entry to Admin Portal reporting section | New `Customer` tab added in `apps/creative-portal/components/pages/reporting/index.tsx`, rendering `ClientUsage` partial | ✅ Addressed |
| Accessible to all authenticated admin users | Page `getServerSideProps` restricts to `Admin`, `ServiceDelivery`, `Superadmin`. **API endpoint does NOT enforce role-based access** | ⚠️ Partial — see Issue #1 |
| Single-select Organization filter (must be selected before data loads) | `OrganizationFilter` (creative-portal), backed by `useAllOrganizationsQuery({ status: "A" })` (Org Search API), gating data fetch via `selectedOrgId` | ✅ Addressed |
| Multi-select Project filter populated from org groups | `ProjectFilter` (shared) uses `useOrgGroups({ orgId })` (Org Group Search API) | ✅ Addressed |
| Project filter default = All; changes when org changes (reset to All) | `handleOrgChange` resets `selectedOrgGroupIds` to `[]`, which is treated as "All" by `activeOrgGroupIds` derivation | ✅ Addressed |
| Charge requests batched per selected org group (Complete only) | `getUsageReportData.ts` calls `fetchOrders` per org group, filters `status === "Complete"`, then fetches charges per order via `mapWithConcurrency` (limit 10) | ✅ Addressed (with caveat — see Issue #5) |
| Single Complete section grouped by project | `SummaryCards` rendered with `showInProgress={false}`; In Progress UI is suppressed in cards | ✅ Addressed (Cards only — chart still shows In Progress; see Issue #2) |
| Summation aligns with PP-1633 customer portal | Re-uses `calculateChargeTotalFromCharges` and `fetchChargesForOrder` (with `isBeforeChargeData`) — same pipeline as `useOrderChargeTotalQuery` | ✅ Addressed |
| If no organization selected → prompt | `OrganizationPrompt` rendered when `!selectedOrgId` | ✅ Addressed |
| If no charge records → "No Usage In This Period" | `SummaryCards` empty state shows this when `completeOrderCount === 0` | ✅ Addressed |
| Customer filter (deferred) | Not implemented | ✅ Correctly omitted (PR description incorrectly mentions a CustomerFilter — see Issue #7) |
| Date Range filter (deferred) | Not used by Admin Client Usage. Component remains shared and consumed only by customer-portal usage report | ✅ Correctly omitted in Admin tab |
| In Progress section (deferred) | Cards suppressed via `showInProgress={false}`; chart legend/tooltip still references it | ⚠️ Partial — see Issue #2 |

**Scope deltas vs Jira:**
- Adds a `MIN_REPORT_DATE_ISO = "2026-02-01"` filter on the server side that drops orders whose earliest charge is before Feb 1, 2026. This mirrors the customer-portal's StartDate semantics and is appropriate even though the Date Range filter is deferred (data quality floor).
- Extracts `SummaryCards`/`UsageChart`/`ProjectFilter`/`DateRangeFilter` into `@proofed/shared` and migrates customer-portal to consume them. This is a legitimate refactor; verify no customer-portal regression (Issue #6).

---

## Architecture Analysis

The PR follows the standard B2B layering: page → partials → hooks/services → API handler → service-layer fetcher. New shared primitives (`SummaryCards`, `UsageChart`, `DateRangeFilter`, `ProjectFilter`, `mapWithConcurrency`, `computeOrderMetrics`, `computeDateRange`, `formatUsageCost`) are extracted cleanly into `packages/shared` and re-consumed by both portals.

Key divergences from customer-portal (PP-1633):
- The Charge service's `SearchBy=organizationGroupId` response omits `orderId`, so the per-group batched-charges shortcut from PP-1633 isn't reusable. The handler instead fans out per-order charge fetches (bounded by `mapWithConcurrency`, limit 10). This is documented at the top of `getUsageReportData.ts`, with a follow-up note to revisit once the Charge API returns `orderId`.
- The admin Client Usage hook uses `isFetching` for `isLoading`; the customer-portal hook uses `isLoading`. This is a subtle UX inconsistency — see Issue #4.

The `withApiMiddleware` defaults (`authNeeded: true`, `requiredRoles: []`) enforce session auth but NOT role-based authorization. The endpoint also does not filter results by the requesting user's accessible organizations.

---

## Issues Found

### 1. New API endpoint has no role-based authorization or data-scope check

**[File: apps/creative-portal/api/mixtures/usageReportData/getUsageReportData.ts]**
**Function/Class:** `getUsageReportData` / `withApiMiddleware(handler, options)`
**Severity:** high
**Problem:** `options` only sets `schema`; it does not pass `requiredRoles`. `withApiMiddleware` defaults `requiredRoles: []`, so any authenticated creative-portal user (specialists, editors, etc.) can call `GET /api/mixtures/usageReportData?organizationGroupIds=…` and receive charge totals for **any** organization group by id. The handler does not cross-check that the caller has access to the requested org groups.
**Impact:** Privilege escalation in the creative portal — non-admin authenticated users can enumerate financial data for arbitrary client organizations. The page is correctly gated to Admin/ServiceDelivery/Superadmin via `getServerSideProps`, but the API surface bypasses that.
**Fix:** Restrict the endpoint to admin-tier roles (matching the page) and document the intent. Example:

```typescript
const options = {
  requiredRoles: [
    UserRole.Admin,
    UserRole.ServiceDelivery,
    UserRole.Superadmin
  ],
  schema: getUsageReportDataSchema
} as const;
```

If a broader audience is intended, the handler should additionally verify the requester has access to each `organizationGroupId` (e.g. via the existing org-membership service) before fetching charges.

### 2. UsageChart legend and tooltip always reference "In Progress" — leaks deferred-scope UI into the Admin Customer tab

**[File: packages/shared/components/molecules/UsageReport/UsageChart/index.tsx, .../partials/CustomTooltip/index.tsx]**
**Function/Class:** `UsageChart`, `CustomTooltip`
**Severity:** high
**Problem:** The Jira reduces admin scope to a single Complete section — In Progress is deferred. `SummaryCards` correctly accepts `showInProgress` and the admin passes `false`. However, `UsageChart` hard-codes both legend entries (`Complete`, `In Progress`) and an `inProgressOrders` bar, and `CustomTooltip` renders an "In Progress" row plus a "% Complete" line and the `IN_PROGRESS_CHART_FOOTER_TEXT` footer ("Hourly charges will not be correctly calculated for 'In Progress' orders.") in every tooltip — including admin tooltips where In Progress is always 0.
**Impact:** Admin users see an "In Progress: $0.00" line and a footer warning about hourly charges that's not relevant to the Complete-only view. The chart legend is misleading. This directly conflicts with the Jira: "The report must display a single Complete section".
**Fix:** Thread a `showInProgress?: boolean` prop (default `true`) through `UsageChart` → `CustomTooltip`, and from `ClientUsage` pass `false`. Conditionally render the In Progress legend item, the `inProgressOrders` `<Bar>`, the In Progress tooltip row, the "% Complete" row, and the footer text. The customer-portal call site keeps the default and behaves unchanged.

### 3. `useOrgGroups` is called with `orgId: 0` when no org is selected

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/hooks.ts]**
**Function/Class:** `useClientUsageReport`
**Severity:** low
**Problem:** `useOrgGroups({ orgId: selectedOrgId ?? 0, options: { enabled: !!selectedOrgId } })`. The query is correctly disabled, but the React Query cache key still includes `orgId.toString() === "0"`. If anything else in the app legitimately calls `useOrgGroups({ orgId: 0 })` it would collide. More importantly, it's a smell — passing a sentinel that the producer treats as valid.
**Impact:** Minor — no current bug, but obscures intent and could cause cache confusion if `useOrgGroups`'s contract changes.
**Fix:** Guard the call, e.g. only render the dependent state when selected, or change the `useOrgGroups` signature to accept `number | null` and short-circuit internally.

### 4. Admin and customer hooks differ on `isLoading` vs `isFetching`

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/hooks.ts]**
**Function/Class:** `useClientUsageReport`
**Severity:** medium
**Problem:** `isLoading = isFetchingOrgGroups || isFetchingReportData`. The customer-portal sibling (`apps/customer-portal/components/pages/reports/usage/hooks.ts`) uses `isLoading` from each query. `isFetching` is `true` on every background refetch (e.g. tab refocus); `isLoading` is only `true` until the first response. With the current admin hook, every refetch will swap the chart for the loader, hiding existing data.
**Impact:** UX regression vs the customer-portal pattern — flicker on background refresh; existing data disappears behind the loader between renders even when there is cached data to show.
**Fix:** Use `isLoading` (not `isFetching`) for both queries to match the customer-portal behavior, unless there is a deliberate reason for the divergence (in which case it should be commented).

```typescript
const {
  data: rawOrgGroups = [],
  isLoading: isLoadingOrgGroups,
  isError: hasOrgGroupsError
} = useOrgGroups({ ... });

const {
  data: usageReportData,
  isLoading: isLoadingReportData,
  isError: hasError
} = useUsageReportDataQuery(...);

const isLoading = isLoadingOrgGroups || isLoadingReportData;
```

### 5. `mapWithConcurrency` rejects on first failure but lets sibling workers run unobserved

**[File: packages/shared/utils/mapWithConcurrency.ts]**
**Function/Class:** `mapWithConcurrency`
**Severity:** low
**Problem:** When one worker's `fn` rejects, `Promise.all(workers)` rejects, but the remaining workers' loops continue executing (`while (nextIndex < items.length)`) until items are exhausted. Their results land in `results` but the outer caller has already moved on. For unbounded server-side fan-out this can amplify load on a failing downstream.
**Impact:** Not exercised in this PR (the per-order async fn catches its own errors and returns `null`, so `mapWithConcurrency` never sees a rejection). But the utility is now in `@proofed/shared` and the next consumer may not pre-catch.
**Fix:** Set a `hasRejected` flag and short-circuit the worker loop, or use an `AbortSignal`. At minimum, document the semantics in a one-line comment so future callers know what to expect.

### 6. Customer-portal `MonthlyUsageData.year` and `groupId`/`groupName` shape change — confirm no downstream regression

**[File: packages/shared/types/usageReport/types.ts, apps/customer-portal/components/pages/reports/usage/hooks.ts]**
**Function/Class:** `MonthlyUsageData`, `useUsageReport` (customer-portal)
**Severity:** medium
**Problem:** `MonthlyUsageData` now has a required `year: number` field that did not previously exist in the customer-portal local types. Customer-portal's hook was rewritten to (a) map `rawOrgGroups` to the new `ProjectGroup` shape (`groupId`/`groupName`) and (b) include `year` on each `monthly` entry. The test fixtures were updated in `index.test.tsx`.
**Impact:** The customer-portal usage report now renders an updated `CustomTooltip` that shows `"{month.slice(0,3)} {year-2-digits}"` (e.g. "Mar 26") where previously the type only carried `month`. If the tooltip previously read `year` from elsewhere or this is the first time `year` is shown, it is a visible UX change in the customer portal — please confirm this is intentional / aligns with design.
**Fix:** Visually QA the customer-portal usage report tooltip and confirm "Mar 26"-style display is intended (and ideally screenshot-attached to the PR), or revert the tooltip to month-only for parity.

### 7. PR description claims a `CustomerFilter` was added; the code does not contain one

**[File: PR description]**
**Function/Class:** N/A
**Severity:** low
**Problem:** The PR description's "Areas of Change" lists `apps/creative-portal/components/pages/reporting/partials/ClientUsage/` "new UI (OrganizationFilter, **CustomerFilter**, hooks, styles)". The repo has only `OrganizationFilter/` — there is no `CustomerFilter/` directory. Customer filter is deferred per Jira.
**Impact:** Reviewers and future archaeologists will be misled. Doesn't change behavior.
**Fix:** Update the PR description to drop the `CustomerFilter` reference and explicitly call out that Customer filter, Date Range filter (in the admin tab), and In Progress section are deferred per the 2026-04-17 scope reduction.

### 8. `ClientUsage` index has no rendering/UI test

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx]**
**Function/Class:** `ClientUsage`
**Severity:** medium
**Problem:** `hooks.ts`, `OrganizationFilter`, `getUsageReportData`, `fetchChargesForOrder`, and shared utilities all have tests. The composing component itself (which is responsible for the "Please select an organization" prompt, the error banner, the conditional rendering of `SummaryCards`/`UsageChart`, and the `LoadingWrapper` orchestration) has no test. Per CLAUDE.md, "Every PR must include tests for new code".
**Impact:** Future refactors of the conditional branches in `ClientUsage` can silently regress the empty/error/loading UI without test failures.
**Fix:** Add `index.test.tsx` covering: (a) no-org prompt shown when `selectedOrgId === null`; (b) error banner shown when query errors; (c) loader shown when `isLoading`; (d) chart hidden when `monthlyData.length === 0` but summary still rendered.

### 9. JSX prop forwarding doesn't follow the spread-with-object-literal convention

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx]**
**Function/Class:** `ClientUsage`
**Severity:** low
**Problem:** Per CLAUDE.md: "When a parent forwards several local variables as props, prefer the spread-with-object-literal pattern over a sequence of explicit `prop={local}` assignments". `<OrganizationFilter isLoading={isLoading} onOrgChange={handleOrgChange} selectedOrgId={selectedOrgId} />` and `<ProjectFilter orgGroups={orgGroups} selectedOrgGroupIds={selectedOrgGroupIds} onSelectionChange={setSelectedOrgGroupIds} />` are exactly the case the rule targets.
**Impact:** Style only.
**Fix:**

```tsx
<OrganizationFilter
  {...{ isLoading, onOrgChange: handleOrgChange, selectedOrgId }}
/>
<ProjectFilter
  {...{
    orgGroups,
    selectedOrgGroupIds,
    onSelectionChange: setSelectedOrgGroupIds
  }}
/>
```

### 10. `OrganizationPrompt` doubles as the loading skeleton container

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx, styles.ts]**
**Function/Class:** `ClientUsage`
**Severity:** low
**Problem:** The `LoadingWrapper`'s `skeleton` reuses `<OrganizationPrompt>` (which is semantically "select an organization" copy + height) as a container for `<Loader />`. It works visually because both are full-height centered flexes, but the styled-component name now lies — `OrganizationPrompt` is also used as a generic centered-loader frame.
**Impact:** Minor maintainability — renaming or restyling `OrganizationPrompt` will silently affect the loading state.
**Fix:** Extract a `CenteredLoader` (mirroring `apps/customer-portal/components/pages/reports/styles.ts`) and use it for the loader; keep `OrganizationPrompt` for the prompt text only.

---

## Tests

- ✅ `getUsageReportData.test.ts` — comprehensive: order bucketing by earliest charge timestamp, MIN_DATE filter, dedupe, status filter, concurrency cap, charge-fetch failure, empty input.
- ✅ `fetchChargesForOrder.test.ts` — covers `isBeforeChargeData` branches, minimumChargeAmount fallback (including 0 vs undefined), failure paths, words-unit detection.
- ✅ `ClientUsage/hooks.test.ts` — covers initial state, org-change reset, fallback to all org-group ids, explicit-selection, isFetching propagation, summary/monthly derivation.
- ✅ `OrganizationFilter/index.test.tsx` — covers query call, options, placeholder, selected value, onChange (numeric id + null), disabled.
- ✅ `packages/shared/utils/mapWithConcurrency.test.ts` — order preservation, concurrency cap, empty input, limit-exceeds-items, rejection.
- ✅ `packages/shared/utils/usageReport/computeDateRange.test.ts`, `computeOrderMetrics.test.ts`, `calculateOrderTotal.test.ts`, `formatUsageCost.test.ts` — moved + updated.
- ✅ `packages/shared/components/molecules/UsageReport/SummaryCards/SummaryCards.test.tsx`, `UsageChart/utils.test.ts`, `UsageChart/partials/CustomTooltip/CustomTooltip.test.tsx`, `ProjectFilter/ProjectFilter.test.tsx` — added or moved.
- ❌ No UI test for the composing `ClientUsage/index.tsx` (Issue #8).
- ⚠️ Customer-portal `index.test.tsx` updated for new `MonthlyUsageData.year` field and new `ProjectGroup` shape — confirm visually that the tooltip behavior matches design (Issue #6).
- ⚠️ Manual testing checkbox in PR is unchecked. Per CLAUDE.md Figma/UI rule, the admin Customer tab should be visually compared to the Figma node before merging.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Mostly — chart UI leaks In Progress into Admin tab (#2) |
| Regression risk | ⚠️ Medium — customer-portal usage tooltip behavior changed (#6); API authz gap (#1) |
| Tests | ⚠️ Good coverage on hooks/utils/handlers; missing UI test for `ClientUsage/index.tsx` |
| Code quality | ✅ Strong — clean shared/portals split, well-commented handler trade-offs, isolated utilities |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes**

Must-fix before merge:
1. Add `requiredRoles` (Admin / ServiceDelivery / Superadmin) to the new `/api/mixtures/usageReportData` handler, mirroring the page-level guard. (Issue #1)
2. Suppress the "In Progress" legend, bar, tooltip row, "% Complete" row, and footer in `UsageChart` / `CustomTooltip` when In Progress is out of scope (thread `showInProgress` through). (Issue #2)
3. Confirm intentional behavior change in customer-portal usage tooltip (new "Mar 26"-style year display). Visual QA + screenshot in PR. (Issue #6)

Should-fix:
4. Swap `isFetching` → `isLoading` in the admin hook to match customer-portal UX. (Issue #4)
5. Add `ClientUsage/index.test.tsx` covering prompt/error/loading/empty branches. (Issue #8)
6. Update PR description to remove the inaccurate "CustomerFilter" mention and call out deferred scope. (Issue #7)

Nice-to-have:
7. Tidy `useOrgGroups` call to avoid sentinel-`0`. (Issue #3)
8. Apply the spread prop-forwarding convention in `ClientUsage/index.tsx`. (Issue #9)
9. Extract a `CenteredLoader` style instead of reusing `OrganizationPrompt` as a loader frame. (Issue #10)
10. Document/short-circuit `mapWithConcurrency` rejection semantics. (Issue #5)
