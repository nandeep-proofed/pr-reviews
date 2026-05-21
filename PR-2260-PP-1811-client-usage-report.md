# PR Review: feature/PP-1811: Add Client Usage Report to Admin Portal reporting section

**PR:** https://github.com/Proofed/B2BWebserver/pull/2260
**Jira:** https://proofed.atlassian.net/browse/PP-1811
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Add "Usage" entry to Admin Portal reporting section | New `Customer` tab in `apps/creative-portal/components/pages/reporting/index.tsx`, rendering `ClientUsage` partial | ✅ Addressed |
| Accessible to all authenticated admin users | Page `getServerSideProps` restricts to Admin / ServiceDelivery / Superadmin. **API endpoint does NOT enforce roles** | ⚠️ Partial — see Issue #1 |
| Single-select Organization filter; must be selected before data loads | `OrganizationFilter` + `useAllOrganizationsQuery({ status: "A" })` (Org Search API); data query disabled until `selectedOrgId` is set | ✅ Addressed |
| Multi-select Project filter populated from org groups | Shared `ProjectFilter` + `useOrgGroups({ orgId })` (Org Group Search API) | ✅ Addressed |
| Project default = All; changing org resets to All | `handleOrgChange` resets `selectedOrgGroupIds` to `[]`, treated as "All" by `activeOrgGroupIds` | ✅ Addressed |
| Charge requests batched per org group (Complete only) | `fetchOrders` per group, `status === "Complete"` filter, charges fetched via `mapWithConcurrency` (limit 10) | ✅ Addressed (different shape — see Issue #3) |
| Single Complete section grouped by project | `SummaryCards` with `showInProgress={false}` — cards are Complete-only | ⚠️ Partial — chart legend/tooltip still reference In Progress (Issue #2) |
| Summation aligns with PP-1633 | Reuses `calculateChargeTotalFromCharges` + `fetchChargesForOrder(isBeforeChargeData)` — same pipeline as `useOrderChargeTotalQuery` | ✅ Addressed |
| No org selected → prompt | `OrganizationPrompt` rendered when `!selectedOrgId` | ✅ Addressed |
| No data → "No Usage In This Period" | `SummaryCards` empty state when `completeOrderCount === 0` | ✅ Addressed |
| Customer filter (deferred) | Not implemented (PR description incorrectly mentions a `CustomerFilter` — see Issue #10) | ✅ Correctly omitted |
| Date Range filter (deferred) | Not used in Admin tab; component remains shared and consumed only by customer-portal usage report | ✅ Correctly omitted from Admin |
| In Progress section (deferred) | Cards suppressed via `showInProgress={false}`; chart legend/tooltip still reference In Progress | ⚠️ Partial — Issue #2 |

**Scope deltas vs Jira:**
- Server-side `MIN_REPORT_DATE_ISO = "2026-02-01"` floor drops orders whose earliest charge is before Feb 1, 2026. Mirrors the customer-portal's `StartDate` semantics and is appropriate as a data-quality floor even though the Date Range filter is deferred.
- Extracts `SummaryCards` / `UsageChart` / `ProjectFilter` / `DateRangeFilter` / `mapWithConcurrency` / `computeOrderMetrics` / `computeDateRange` / `formatUsageCost` / `calculateOrderTotal` into `@proofed/shared` and migrates customer-portal to consume them — legitimate enabling refactor; regression risk covered below.

---

## Architecture Analysis

Clean shared/portals split — reusable UI primitives and utilities moved into `@proofed/shared`, customer-portal updated to consume them, admin-portal builds on the same shared building blocks. Component structure follows project conventions (`index.tsx` + `hooks.ts` + `styles.ts` + `types.ts` per folder).

Key intentional divergence from PP-1633: the Charge service's `SearchBy=organizationGroupId` response omits `orderId`, so the per-group batched-charges shortcut from PP-1633 isn't reusable in the admin handler. Instead, the handler fans out per-order charge fetches bounded by `mapWithConcurrency` (limit 10), with the trade-off documented at the top of `getUsageReportData.ts` and a noted follow-up to revisit once the Charge API returns `orderId`.

`mapWithConcurrency` is a clean, well-tested concurrency limiter that avoids pulling in a dependency like `p-limit`.

---

## Issues Found

### 1. New API endpoint has no role-based authorization or data-scope check

**[File: apps/creative-portal/api/mixtures/usageReportData/getUsageReportData.ts]**
**Function/Class:** `getUsageReportData` / `withApiMiddleware(handler, options)`
**Severity:** high
**Problem:** `options` only sets `schema`; it doesn't pass `requiredRoles`. `withApiMiddleware` defaults `requiredRoles: []`, so any authenticated creative-portal user (specialists, editors, etc.) can call `GET /api/mixtures/usageReportData?organizationGroupIds=…` and receive charge totals for **any** org group by id. The handler also doesn't cross-check that the caller has access to the requested org groups.
**Impact:** Privilege escalation in the creative portal — non-admin authenticated users can enumerate financial data for arbitrary client organizations. The page is correctly gated via `getServerSideProps`, but the API surface bypasses that.
**Fix:** Restrict the endpoint to admin-tier roles, mirroring the page:

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

If a broader audience is intended, also verify the requester has access to each `organizationGroupId` before fetching charges.

### 2. UsageChart legend and tooltip always reference "In Progress" — leaks deferred-scope UI into the Admin tab

**[File: packages/shared/components/molecules/UsageReport/UsageChart/index.tsx, .../partials/CustomTooltip/index.tsx]**
**Function/Class:** `UsageChart`, `CustomTooltip`
**Severity:** high
**Problem:** Jira reduces admin scope to a single Complete section — In Progress is deferred. `SummaryCards` correctly accepts `showInProgress` and admin passes `false`. But `UsageChart` hard-codes both legend entries (`Complete`, `In Progress`), renders an `inProgressOrders` `<Bar>`, and `CustomTooltip` renders an "In Progress" row, a "% Complete" row, and the `IN_PROGRESS_CHART_FOOTER_TEXT` footer ("Hourly charges will not be correctly calculated for 'In Progress' orders.") on every tooltip — even when `inProgressOrders` is always 0.
**Impact:** Admin users see an "In Progress: $0.00" line on every tooltip and an irrelevant footer warning about hourly charges. The chart legend is misleading. Directly conflicts with "The report must display a single Complete section".
**Fix:** Thread a `showInProgress?: boolean` prop (default `true`) through `UsageChart` → `CustomTooltip`. From `ClientUsage` pass `false`. Conditionally render the In Progress legend item, the `inProgressOrders` `<Bar>`, the In Progress tooltip row, the "% Complete" row, and the footer text. Customer-portal call sites keep the default and behave unchanged.

### 3. Order fan-out is unbounded and pre-dates the MIN_REPORT_DATE filter

**[File: apps/creative-portal/api/mixtures/usageReportData/getUsageReportData.ts]**
**Function/Class:** `getUsageReportData`
**Severity:** medium
**Problem:** Two compounding cost issues:
1. `fetchOrders({ ..., includeClosed: true })` retrieves ALL completed orders for each selected org group, regardless of age. The `MIN_REPORT_DATE_ISO` floor is applied *after* fetching orders AND charges. For an org with 5 years of history, every order is fetched and every charge is fetched before most are discarded.
2. Charge fetches are issued per-order (because `SearchBy=organizationGroupId` omits `orderId`), bounded by concurrency 10 but with no overall cap. A partner with thousands of complete orders will issue thousands of HTTP requests to the charge service per report load.
**Impact:** High latency and disproportionate load on the orders/charge services for large or long-lived organizations. Risk of request timeouts at scale.
**Fix:**
- If `fetchOrders` accepts a date filter, push a lower-bound (e.g. `MIN_REPORT_DATE_ISO` minus a buffer) into the call so the orders list is pre-trimmed.
- Add an info-level log of the order count and a warning above a threshold (e.g. 500) so this is observable in production.
- The existing code comment notes the follow-up to extend the Charge API to return `orderId` — confirm a ticket exists.

### 4. Admin and customer hooks differ on `isLoading` vs `isFetching`

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/hooks.ts]**
**Function/Class:** `useClientUsageReport`
**Severity:** medium
**Problem:** `isLoading = isFetchingOrgGroups || isFetchingReportData`. The customer-portal sibling (`apps/customer-portal/components/pages/reports/usage/hooks.ts`) uses `isLoading` from each query. `isFetching` is `true` on every background refetch (tab refocus, window focus, refetch-on-mount); `isLoading` is only `true` until the first response.
**Impact:** UX regression vs the customer-portal pattern — existing data is hidden behind the loader on every background refresh, causing flicker even when cached data is available.
**Fix:** Use `isLoading` from each query, matching customer-portal:

```typescript
const { data: rawOrgGroups = [], isLoading: isLoadingOrgGroups, isError: hasOrgGroupsError } = useOrgGroups({ ... });
const { data: usageReportData, isLoading: isLoadingReportData, isError: hasError } = useUsageReportDataQuery(...);
const isLoading = isLoadingOrgGroups || isLoadingReportData;
```

### 5. Customer-portal `MonthlyUsageData.year` introduces a visible tooltip change

**[File: packages/shared/types/usageReport/types.ts, apps/customer-portal/components/pages/reports/usage/hooks.ts, .../UsageChart/partials/CustomTooltip/index.tsx]**
**Function/Class:** `MonthlyUsageData`, customer-portal `useUsageReport`, `CustomTooltip`
**Severity:** medium
**Problem:** `MonthlyUsageData` now has a required `year: number` field that didn't exist in the previous customer-portal local type. Customer-portal's hook was rewritten to (a) map `rawOrgGroups` to the new `ProjectGroup` shape (`groupId`/`groupName`) and (b) include `year` on each `monthly` entry. The shared `CustomTooltip` now renders `"{month.slice(0,3)} {String(year).slice(-2)}"` (e.g. "Mar 26"). The customer-portal `index.test.tsx` was updated for the new shape.
**Impact:** Visible UX change in the customer-portal usage report tooltip — previously month-only labels, now month + 2-digit year. May or may not be intended.
**Fix:** Visually QA the customer-portal tooltip and confirm "Mar 26"-style display matches the design (screenshot in PR), or revert the tooltip to month-only for parity. Also add a hooks-level test in `apps/customer-portal/components/pages/reports/usage/hooks.test.ts` that the `year` field is correctly derived from order dates spanning multiple years — current tests pre-supply `year` in fixtures, so the parsing path is untested.

### 6. `ClientUsage/index.tsx` has no UI test

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx]**
**Function/Class:** `ClientUsage`
**Severity:** medium
**Problem:** Hooks, `OrganizationFilter`, handler, and shared utilities all have tests. The composing component (which orchestrates the "Please select an organization" prompt, the error banner, the loader, and the chart/cards visibility) is not directly tested. CLAUDE.md: "Every PR must include tests for new code".
**Impact:** Conditional rendering branches in `ClientUsage` can silently regress without test failures.
**Fix:** Add `ClientUsage/index.test.tsx` covering: (a) no-org prompt when `selectedOrgId === null`; (b) error banner when query errors; (c) loader when `isLoading`; (d) chart hidden when `monthlyData.length === 0` while cards still render.

### 7. `OrganizationFilter/hooks.ts` has no dedicated test

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/partials/OrganizationFilter/hooks.ts]**
**Function/Class:** `useOrganizationFilter`
**Severity:** medium
**Problem:** The hook contains non-trivial logic — option mapping from organizations, selected-value memoization, and the change handler with type narrowing between `SingleValue` and `MultiValue`. The component test (`index.test.tsx`) exercises it through the integration but doesn't isolate the hook.
**Impact:** Hook-level logic changes (e.g. change-handler edge cases like clearing while selection is already null) could regress undetected.
**Fix:** Add `hooks.test.ts` directly testing option building, selected-value derivation, and the change handler callback (including the `null` and `MultiValue` defensive branches).

### 8. `mapWithConcurrency` rejects on first failure but lets sibling workers run unobserved

**[File: packages/shared/utils/mapWithConcurrency.ts]**
**Function/Class:** `mapWithConcurrency`
**Severity:** low
**Problem:** When one worker's `fn` rejects, `Promise.all(workers)` rejects, but the remaining workers' `while (nextIndex < items.length)` loops continue until items are exhausted. Their results land in `results` but the caller has already moved on.
**Impact:** Not exercised in this PR (the per-order async fn catches its own errors and returns `null`, so `mapWithConcurrency` never sees a rejection). But the utility is now in `@proofed/shared` and the next consumer may not pre-catch — risk of amplifying load on a failing downstream.
**Fix:** Set a `hasRejected` flag and short-circuit the worker loop, or accept an `AbortSignal`. At minimum, document the semantics in a one-line comment so future callers know what to expect.

### 9. `useOrgGroups` is called with `orgId: 0` when no org is selected

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/hooks.ts]**
**Function/Class:** `useClientUsageReport`
**Severity:** low
**Problem:** `useOrgGroups({ orgId: selectedOrgId ?? 0, options: { enabled: !!selectedOrgId } })`. The query is correctly disabled, but the React Query cache key still includes `orgId.toString() === "0"`. Sentinel-`0` is treated as if it were a valid org id by the producer.
**Impact:** No current bug, but obscures intent and could cause cache confusion if `useOrgGroups`'s contract changes.
**Fix:** Guard the call (e.g. only render dependents when selected) or change `useOrgGroups` to accept `number | null` and short-circuit internally.

### 10. PR description claims a `CustomerFilter` was added; the code has none

**[File: PR description]**
**Severity:** low
**Problem:** The PR description's "Areas of Change" lists `apps/creative-portal/components/pages/reporting/partials/ClientUsage/` "new UI (OrganizationFilter, **CustomerFilter**, hooks, styles)". The repo has only `OrganizationFilter/` — there is no `CustomerFilter/`. Customer filter is deferred per Jira.
**Impact:** Misleads reviewers and future archaeologists. No behavioral effect.
**Fix:** Update the description: drop the `CustomerFilter` reference and explicitly call out that Customer filter, Date Range filter (in the admin tab), and In Progress section are deferred per the 2026-04-17 scope reduction.

### 11. JSX prop forwarding doesn't follow the spread-with-object-literal convention

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx]**
**Function/Class:** `ClientUsage`
**Severity:** low
**Problem:** Per CLAUDE.md: "When a parent forwards several local variables as props, prefer the spread-with-object-literal pattern over a sequence of explicit `prop={local}` assignments". Both child renders are exactly this case.
**Impact:** Style only.
**Fix:**

```tsx
<OrganizationFilter
  {...{ isLoading, onOrgChange: handleOrgChange, selectedOrgId }}
/>
<ProjectFilter
  {...{ orgGroups, selectedOrgGroupIds, onSelectionChange: setSelectedOrgGroupIds }}
/>
```

### 12. `OrganizationPrompt` doubles as the loading skeleton container

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx, styles.ts]**
**Function/Class:** `ClientUsage`
**Severity:** low
**Problem:** `LoadingWrapper`'s `skeleton` wraps `<Loader />` inside `<OrganizationPrompt>` (which is the styled-component for the "select an organization" text frame). Works visually because both are full-height centered flexes, but the name no longer matches its role.
**Impact:** Maintainability — restyling `OrganizationPrompt` will silently affect the loader.
**Fix:** Extract a `CenteredLoader` (mirroring `apps/customer-portal/components/pages/reports/styles.ts`) for the loader; keep `OrganizationPrompt` for the prompt text only.

### 13. Redundant `|| false` in customer-portal `useUsageReportDataQuery.enabled`

**[File: apps/customer-portal/services/usageReportData/index.ts]**
**Function/Class:** `useUsageReportDataQuery`
**Severity:** low
**Problem:** The customer-portal service hook has:

```typescript
enabled:
  (options?.enabled !== false &&
    !!params &&
    params.organizationGroupIds.length > 0 &&
    !!params.startDate &&
    !!params.endDate) ||
  false
```

The trailing `|| false` is redundant — the inner expression is already a boolean. (The creative-portal sibling hook doesn't have this.)
**Impact:** Readability only.
**Fix:** Drop `|| false`.

### 14. `MIN_REPORT_DATE_ISO` comparison relies on lexicographic string ordering

**[File: apps/creative-portal/api/mixtures/usageReportData/getUsageReportData.ts]**
**Function/Class:** `getUsageReportData`
**Severity:** low
**Problem:** `earliestChargeTimestamp.split("T")[0] < MIN_REPORT_DATE_ISO` relies on the fact that `YYYY-MM-DD` sorts lexicographically in chronological order. The code comments call this out, which is good, but the pattern is non-obvious and silently breaks if a timestamp ever arrives without the ISO date prefix.
**Impact:** Low — ISO format is stable. Documentation-only concern.
**Fix:** Optional — convert to a Date comparison for clarity:

```typescript
const chargeDate = new Date(earliestChargeTimestamp);
const minDate = new Date(MIN_REPORT_DATE_ISO);
if (chargeDate < minDate) return null;
```

### 15. `CustomTooltip` uses `pastelGreen1` while bars and cards use `green1`

**[File: packages/shared/components/molecules/UsageReport/UsageChart/partials/CustomTooltip/index.tsx]**
**Function/Class:** `CustomTooltip`
**Severity:** low
**Problem:** The tooltip's "Complete" dot uses `theme.colors.pastelGreen1`, but `SummaryCards`'s Complete accent and the chart's `completeOrders` bar use `theme.colors.green1`. Three different presentations of "Complete" in the same view.
**Impact:** Minor visual inconsistency.
**Fix:** Use `theme.colors.green1` in the tooltip for consistency, or add a one-line comment if `pastelGreen1` is intentionally chosen for tooltip legibility.

### 16. Hook return destructuring in `ClientUsage/index.tsx` isn't alphabetical

**[File: apps/creative-portal/components/pages/reporting/partials/ClientUsage/index.tsx]**
**Function/Class:** `ClientUsage`
**Severity:** low
**Problem:** Destructuring order: `summary, monthlyData, orgGroups, selectedOrgId, handleOrgChange, selectedOrgGroupIds, setSelectedOrgGroupIds, isLoading, hasError, hasOrgGroupsError`. Per CLAUDE.md, alphabetical destructuring is preferred (and enforced by `eslint-plugin-perfectionist` for props interfaces).
**Impact:** Style only.
**Fix:** Reorder both the hook return and the destructuring alphabetically.

### 17. Per-portal `fetchChargesForOrder` divergence

**[File: apps/creative-portal/api/utils/charges/fetchChargesForOrder.ts]**
**Function/Class:** `fetchChargesForOrder`
**Severity:** low
**Problem:** The creative-portal `fetchChargesForOrder` calls `enhanceTotalChargeEntry({ ..., isClientApi: false })` and includes the `isBeforeChargeData` enhancement (minimum-charge fallback via `fetchOrderById`, synthesized total charge when none exists, words-unit detection). The customer-portal sibling presumably passes `isClientApi: true` and may have different enhancement behavior.
**Impact:** Two portals could produce different totals for the same order if the enhancement logic drifts further apart.
**Fix:** Document the intentional `isClientApi` divergence at the function level, and consider extracting the shared enhancement pipeline into `@proofed/shared` if it must remain bit-identical across portals.

### 18. `calculateOrderTotal` lost the "does not derive fee from rate fields" test in the move

**[File: packages/shared/utils/usageReport/calculateOrderTotal.test.ts]**
**Function/Class:** `calculateOrderTotal` test
**Severity:** low
**Problem:** Develop's `apps/customer-portal/utils/calculateOrderTotal.test.ts` includes a test "does not derive fee from rate fields" (around line 496) that verified `processingFeeRate` / `processingFeeQuantity` / `processingFeeUnit` are NOT used to derive the processing fee. After the move to shared, the test is gone. The new `OrderDetailForCalculation` type no longer carries those fields, but the safety guarantee they tested is still worth a regression test.
**Impact:** If someone later widens `OrderDetailForCalculation` or re-introduces rate-based derivation, there's no test guarding the prior behavior.
**Fix:** Port the test to the shared file using the current `OrderDetailForCalculation` shape (assert that adding noise fields to `orderDetail` doesn't change `processingFee`), or add a one-line comment in the new test file explaining why the test was intentionally dropped.

---

## Tests

- ✅ `getUsageReportData.test.ts` (creative-portal) — bucketing by earliest charge timestamp, MIN_DATE filter, dedupe, status filter, concurrency cap, charge-fetch failure, empty input. Comprehensive.
- ✅ `fetchChargesForOrder.test.ts` — `isBeforeChargeData` branches, minimumChargeAmount fallback (0 vs undefined), synthesized total, error resilience, logger integration, words-unit detection.
- ✅ `ClientUsage/hooks.test.ts` — initial state, org-change reset, fallback to all groups, explicit selection, isFetching propagation, summary/monthly derivation.
- ✅ `OrganizationFilter/index.test.tsx` — query, options, placeholder, selected value, change (numeric id + null), disabled state.
- ✅ Shared utilities: `mapWithConcurrency`, `computeDateRange`, `computeOrderMetrics`, `calculateOrderTotal`, `formatUsageCost`, `UsageChart/utils`.
- ✅ Shared components: `SummaryCards`, `CustomTooltip`, `ProjectFilter`.
- ✅ Customer-portal `getUsageReportData.test.ts` added for the existing endpoint.
- ❌ No UI test for `ClientUsage/index.tsx` (Issue #6).
- ❌ No hook-level test for `OrganizationFilter/hooks.ts` (Issue #7).
- ⚠️ Customer-portal `hooks.test.ts` doesn't exercise the `year` parsing path — fixtures pre-supply `year` (related to Issue #5).
- ⚠️ Visual QA of the customer-portal tooltip + admin Customer tab vs Figma is not evidenced in the PR (manual-testing checkbox unchecked).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Chart UI leaks deferred In Progress section into Admin tab (#2) |
| Regression risk | ⚠️ Medium — customer-portal tooltip behavior changed (#5); API authorization gap (#1); per-org cost at scale (#3) |
| Tests | ⚠️ Strong on hooks/utils/handlers; missing UI test on `ClientUsage/index.tsx` and hook test on `OrganizationFilter/hooks.ts` |
| Code quality | ✅ Clean shared/portals split, well-commented trade-offs, follows project conventions |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes**

Must-fix before merge:
1. Add `requiredRoles` (Admin / ServiceDelivery / Superadmin) to the new `/api/mixtures/usageReportData` handler. (Issue #1)
2. Suppress the "In Progress" legend / bar / tooltip row / "% Complete" row / footer in `UsageChart` / `CustomTooltip` when out of scope. (Issue #2)
3. Confirm the customer-portal tooltip's new "Mar 26"-style year display is intentional (visual QA + screenshot in PR). (Issue #5)

Should-fix:
4. Push the date floor into `fetchOrders` and log order-count metrics. (Issue #3)
5. Swap `isFetching` → `isLoading` in the admin hook to match customer-portal UX. (Issue #4)
6. Add `ClientUsage/index.test.tsx` and `OrganizationFilter/hooks.test.ts`. (Issues #6, #7)
7. Update the PR description to drop the `CustomerFilter` mention and call out deferred scope. (Issue #10)

Nice-to-have:
8. Tidy `useOrgGroups({ orgId: 0 })` sentinel. (Issue #9)
9. Apply the spread prop-forwarding convention in `ClientUsage/index.tsx`. (Issue #11)
10. Extract a `CenteredLoader` style instead of reusing `OrganizationPrompt`. (Issue #12)
11. Document or short-circuit `mapWithConcurrency` rejection semantics. (Issue #8)
12. Drop redundant `|| false` in customer-portal `useUsageReportDataQuery.enabled`. (Issue #13)
13. Optional Date comparison for clarity on `MIN_REPORT_DATE_ISO`. (Issue #14)
14. Align `CustomTooltip` dot color with bar/card accent (`green1` vs `pastelGreen1`). (Issue #15)
15. Alphabetize hook return + destructuring in `ClientUsage`. (Issue #16)
16. Document the `isClientApi` divergence at `fetchChargesForOrder`. (Issue #17)
17. Port or annotate-out the dropped "does not derive fee from rate fields" `calculateOrderTotal` test. (Issue #18)
