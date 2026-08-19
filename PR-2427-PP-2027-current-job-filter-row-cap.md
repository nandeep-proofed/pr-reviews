# PR Review: fix/PP-2027: Current Job filter — recover orders hidden by the search row cap

**PR:** https://github.com/Proofed/B2BWebserver/pull/2427
**Jira:** https://proofed.atlassian.net/browse/PP-2027
**Status:** Code Review
**Branch:** `fix/PP-2027-current-job-filter-missing-orders` → `develop`
**Size:** 11 files, +531 / −10, 2 commits (supersedes #2426)
**CI:** **no checks reported on this branch.** The Codex bot review also aborted ("usage limits reached"). Nothing automated has validated this PR.
**Validation suite:** Skipped — user opted out. `test` / `typecheck` / `lint` / `build` were **not** run.

---

## What this means for users (non-technical summary)

1. **A backend password is written into our logging system every time an order search fails.** The fan-out this PR adds contacts one partner at a time — around 108 calls per dashboard load. Whenever any one of those calls fails, the service credential used to make it is recorded in the log. Anyone with log access can read it, and it stays for the full log retention period. This is the single thing that must be fixed before merge. It is not visible to end users, but it is the highest-severity item here.
2. **The dashboard can now fail outright where it previously showed a partial list.** The new code asks a second system for the list of partners before it can do its work, and if that system is down the whole page errors instead of showing the orders it already had in hand. The PR's own design note says one failure must not take the dashboard down — that rule was applied to the per-partner calls but not to this one.
3. **Orders can silently vanish from the list, with no error shown.** If one partner's search fails, that partner's orders are dropped and the page still reports success. When looking at completed or cancelled orders with more than one status ticked, a single failing status call discards that partner's *successful* results too. This is the same class of problem the ticket is about — orders missing with no signal — arriving through a different door.
4. **The riskiest path in the change is not covered by tests.** Testing proved that the wiring for completed/cancelled order searches can be broken outright — all of it removed — and every one of the 62 tests still passes. If that wiring breaks, people looking at a past month would see the wrong orders or none, and nothing would catch it.
5. **The core fix works, and the ticket's own diagnosis was wrong.** The author disproved the "job type matching" theory with real measurements and found the actual cause: only the oldest 500 orders were ever being fetched. That investigation is the strongest part of this PR, and the fix does recover the missing orders.

One claim in the PR description does not hold up: the "Services column" is listed as a fixed surface, but that column is drawn from a separate lookup and never used the code that was changed. See Issue 11.

---

## Jira Requirements vs Implementation

PP-2027 is a bug report, not a spec. It has no numbered acceptance criteria — it has a stated cause, a reproduction, and an expected result.

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| **Stated cause:** "excludes matching orders due to string-based job type matching" | **Disproved with evidence.** Instrumented the endpoint on B2B test; the filter matched the reported order correctly (`"wouldMatchFilter": true`) — it was never fetched. Root cause is the 500-row search cap. | ✅ Addressed — and correctly re-diagnosed |
| **Expected result:** Current Job filter returns orders whose active job matches the selected value | Cap-truncated searches now re-run scoped per organization and are unioned + de-duplicated, so previously-unfetched orders reach the filter | ✅ Addressed |
| **Reproduces across all job types** (Service, Review, AI…) | Consistent with the cap theory — a row cap is job-type-agnostic, which is exactly why every job type reproduced | ✅ Explained |
| Steps 1–5 (open order, note active job, filter dashboard by it) | Manual verification of order 21341 under the Review filter is **unticked** in the PR checklist | ⚠️ Partial — not yet confirmed |
| *(not in ticket)* "Current Job not ready" classified as Service | Fixed at the single shared entry point so filter / sort / column agree | ✅ Bonus fix, in scope |
| *(not in ticket)* `getOrganizations` crashed when `options` omitted | `removeEmptyKeys(options ?? {})` | ✅ Bonus fix — required by the new call site |

**Scope assessment.** Two commits, both on-ticket. The `getOrganizations` fix is a genuine prerequisite (the new call site is the first caller to omit `options`), not scope creep. The `mapWithConcurrency` throttle on the order-group fan-out is defensive follow-on from the corpus growth. Nothing here is unrelated.

**Related ticket already split out correctly:** the "Order Status alone returns nothing" behaviour surfaced during testing was raised as PP-2083 rather than folded in. Good discipline.

---

## Architecture Analysis

The change has two independent halves.

**Half 1 — cap recovery (`api/mixtures/orders/searchOrdersForTable/`).** `fetchFilteredOrders` previously issued one unscoped `fetchOrders` and returned it. It now treats a result of exactly `ORDER_SEARCH_ROW_CAP` (500) as evidence of truncation and fans out one search per organization via `getOrganizations` + `mapWithConcurrency`, unioning with `uniqBy(..., "id")`. Reusing `fetchOrdersIfIdentifiersExist` (rather than `fetchOrders` directly) to preserve reporting-service routing is the right call and is well-reasoned in the JSDoc.

The design instinct is sound. The problem is that the resilience the author designed for is applied *inside* the loop but not *around* it, and the failure unit inside `fetchOrdersIfIdentifiersExist` (`Promise.all` over identifier × status) is finer-grained than the `catch` that guards it. The result is a function that is careful about the failure it anticipated and unguarded against the two adjacent ones.

**Half 2 — no-current-job classification (`api/orders/`).** A single early return in `extractCurrentJobFromLiveStatusField` before both the lookup and the `JobType.SERVICE` fallback. Placing it at the one shared entry point is exactly right — the filter, the sort and the column cannot now disagree. This half is clean, minimal and well-tested.

**Reuse discipline: verified good.** `mapWithConcurrency` already exists on `develop` and is reused rather than reimplemented. The hand-rolled try/catch inside it follows that util's own documented guidance ("wrap `fn` in try/catch and return a sentinel"). `ORDER_SEARCH_CONCURRENCY` mirrors the established per-fan-out pattern (`CHARGE_FETCH_CONCURRENCY` in `usageReportData`). `NO_CURRENT_JOB_PREFIX` is defined once and used once — no divergent hardcoding anywhere in either app. Interface member ordering, import ordering, Prettier and file/folder conventions were all checked against `packages/eslint-config` and are **clean**.

---

## Issues Found

### 1. Backend service credential is written into the log stream on every upstream failure

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:127-134]**

> **In plain terms:** Whenever the system fails to fetch orders for one partner, it writes the password it used for that request into our logging system in plain text. The new code makes roughly 108 such requests per dashboard load, refreshed every 30 seconds, so it only takes one flaky partner for the credential to be recorded — repeatedly. Anyone who can read logs can read the password, and rotating it means touching the services registry.

**Function/Class:** `fetchOrdersForAllOrganizations`

**Severity:** blocker

**Confidence:** high — reproduced directly

**How to spot it:** Not user-reproducible. Trigger any upstream 5xx/timeout from the orders or order-reporting service during the fan-out, then inspect the Logtail/BetterStack event for `"Failed to search orders for organization"` and read `fields.error.config.headers`.

**Problem:** The raw error object is passed as a structured log field. For an `AxiosError` this serialises the entire request config, including headers.

**Evidence:** The offending code:

```ts
} catch (error) {
  logger.warn("Failed to search orders for organization", {
    organizationId,
    error
  });
```

The chain, each link read: `packages/shared/api/services/getServiceMetadata.ts:19-22` builds `serviceHeaders = { "api-key": service.api_key, Version: service.version }`; `prepareServiceAxiosConfigs.ts:47-52` merges it into the axios config; `@logtail/next` serialises log fields with `JSON.parse(JSON.stringify(args, jsonFriendlyErrorReplacer))` (`node_modules/@logtail/next/dist/logger.js:144`). Per the JSON spec, `toJSON()` runs **before** the replacer, so `AxiosError.toJSON()` (`node_modules/axios/lib/core/AxiosError.js:55-72`) wins and returns `config: utils.toJSONObject(this.config)`. The replacer's `value instanceof Error` guard then sees a plain object and passes it through untouched.

I ran this against the repo's installed axios 1.15.0 with the real replacer:

```
keys: [ 'message', 'name', 'stack', 'config', 'code', 'status' ]
config.headers: {"Accept":"application/json, text/plain, */*",
                 "api-key":"SUPER-SECRET-ORDERS-KEY","Version":"38",
                 "requesterId":1231,"searchBy":"orgId","searchValue":10}
```

The repo explicitly classifies this as a secret: `packages/shared/utils/sentryScrubber.ts:5-11` lists `"api-key"` in `SENSITIVE_FIELDS`. That scrubber is wired to Sentry's `beforeSend` only — **there is no equivalent on the Logtail path.**

**Impact:** A long-lived backend credential plus internal service topology land in the log sink, at a frequency proportional to the fan-out width. Also printed to stdout when `LOGTAIL_SOURCE_TOKEN` is unset (the `prettyPrint` fallback).

**Fix:** Never pass a raw error as a log field. The repo already has both correct forms — `getUsageReportData.ts:102-105` omits the error entirely, and `submitJob.ts:195-202` logs the message only.

```typescript
logger.warn("Failed to search orders for organization", {
  organizationId,
  status: axios.isAxiosError(error) ? error.response?.status : undefined,
  message: error instanceof Error ? error.message : String(error)
});
```

Note the same anti-pattern pre-exists at `api/mixtures/orders/[orderId]/getOrderActivityLog/getOrderActivityLog.ts:56,73`. That is a separate follow-up — but this PR adds a far higher-frequency instance, so fixing it here should not wait on that.

---

### 2. `getOrganizations` sits outside the resilience boundary and hard-fails the dashboard

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:101]**

> **In plain terms:** To recover the missing orders, the page now first asks a different system for the list of partners. If that system is slow or down, the whole dashboard shows an error — even though the orders it needs to display have already been fetched successfully and are sitting in memory. Before this change the page could not fail this way; it showed a shortened list instead. A shortened list is much better than an error page, and the PR's own design note says exactly that.

**Function/Class:** `fetchOrdersForAllOrganizations`

**Severity:** high

**Confidence:** high — confirmed by adversarial verification

**Steps to reproduce:**

1. Sign in to the creative portal as any user and open the Order Dashboard.
2. Apply a Current Job or Order Status filter but **no** Customer filter (this is the ordinary way the dashboard is used).
3. Have the environment hold enough live orders that the first search returns 500 rows — the production condition this PR exists to handle.
4. Make the organization service unavailable (stop it, or block it at the network).
5. **Expected:** the dashboard renders the orders that were already fetched, ideally with a note that the list is incomplete.
6. **Actual:** HTTP 500 and an error page. The 500 already-fetched orders are discarded.

**Problem:** Every per-organization search is individually caught (`utils.ts:107-134`), but the single call the whole fan-out depends on is not — and it runs *after* `fetchOrders` at `utils.ts:179` has already succeeded.

**Evidence:** `const organizations = await getOrganizations(requesterId, {});` — no `try`, no `.catch()`. Verification traced the full chain for any swallow and found none: `api/organizations/getByName.ts:27` is a bare `await axios.get(...)`; `prepareServiceAxiosConfigs.ts:29` throws `new Error("Service not found")`; `getService.ts:22` throws. `handleEndpointError` (`packages/shared/api/utils/handleEndpointError.ts:31,82`) starts at `let statusCode = 500` and ends `return res.status(statusCode).json(responsePayload)` — no degradation path. The base branch's unscoped branch was a bare `return fetchOrders({...})` with no second upstream dependency, so this is genuinely new.

**Impact:** A strict availability regression that makes the organization service a new hard dependency of the order dashboard. It directly contradicts the function's own JSDoc at `utils.ts:89-91`: *"A single organization's failure must not take the whole dashboard down."*

**Fix:** Apply the design rule the PR already states:

```typescript
let organizations: Organization[] = [];

try {
  organizations = await getOrganizations(requesterId, { status: "A" });
} catch (error) {
  logger.warn("Organization lookup failed; returning capped order list", {
    message: error instanceof Error ? error.message : String(error)
  });

  return [] as OrderFromSearch[];
}
```

Returning `[]` makes `fetchFilteredOrders` fall back to the truncated-but-real 500 rows via its existing `uniqBy` union. (The `status: "A"` addition is Issue 7.)

---

### 3. The closed-order / reporting wiring is entirely unpinned — proven by mutation testing

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:188-196 and :113]**

> **In plain terms:** When someone views completed or cancelled orders for a past month, the recovery code has to be told which month, which statuses, and that it is looking at closed orders. Testing proved that instruction can be removed completely and every single test still passes. If it ever breaks, people reviewing a past month would see the wrong orders — or none at all — and no test would notice.

**Function/Class:** `fetchFilteredOrders` → `fetchOrdersForAllOrganizations`

**Severity:** high

**Confidence:** high — proven by mutation, not inferred

**How to spot it:** Not user-reproducible today; this is a test-coverage defect on the highest-risk path. Verify by mutation: replace the call at `utils.ts:188-196` with `{ hasClosedOrderStatus: false, logger, requesterId }` and run the suite.

**Problem:** Every test in `describe("fetchFilteredOrders — unscoped search")` uses `baseParams()` (`utils.test.ts:40-45`), which hard-codes `hasClosedOrderStatus: false` and passes no `orderStatus` / `searchMonth` / `searchYear`. The reporting-routing test (`utils.test.ts:225-247`) exercises `fetchOrdersForAllOrganizations` **directly**, never through `fetchFilteredOrders` — so the entry point the handler actually calls has zero coverage for the closed-order case.

**Evidence:** Two mutations were applied to the production code and the full suite re-run:

- Replacing the argument object at `utils.ts:188-196` with `{ hasClosedOrderStatus: false, logger, requesterId }` — dropping `orderStatus`, `searchMonth`, `searchYear` and forcing the closed flag off — left **11/11 tests passing**.
- Changing `utils.ts:113` from `includeClosed: hasClosedOrderStatus` to `includeClosed: true` left **11/11 tests passing**.

**Impact:** If that wiring regresses, `shouldUseReporting` (`utils.ts:38-43`) goes false and every fanned-out organization silently queries the *live* orders service instead of the reporting service — wrong rows, or none, for the month being viewed. This is the single most consequential untested path in the change.

**Fix:** Exercise the closed-order case through the real entry point, and assert the live path's options too:

```typescript
it("keeps the reporting routing when a closed-order search hits the cap", async () => {
  mockFetchOrders.mockResolvedValueOnce(makeCappedPage());
  mockGetOrganizations.mockResolvedValueOnce([{ id: 10 }]);
  mockFetchReportingOrders.mockResolvedValue([makeOrder(21341)]);

  const orders = await fetchFilteredOrders({
    ...baseParams(),
    hasClosedOrderStatus: true,
    searchMonth: 8,
    searchYear: 2026,
    orderStatus: ["Complete"]
  });

  expect(mockFetchReportingOrders).toHaveBeenCalledWith(
    expect.objectContaining({
      searchBy: "orgId",
      searchValue: 10,
      searchMonth: 8,
      searchYear: 2026,
      searchStatus: "Complete"
    })
  );
  expect(orders).toEqual(expect.arrayContaining([makeOrder(21341)]));
});
```

---

### 4. One failing status call discards an entire organization's successfully-fetched orders

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:70, with the catch at :127-134]**

> **In plain terms:** When someone ticks both "Complete" and "Cancelled", the system makes two separate requests per partner. If just one of those two fails, the partner's orders from the request that *succeeded* are thrown away as well. The page still reports success, so the user sees a shorter list with no indication anything went wrong — and because the dashboard refreshes every 30 seconds, rows can appear and disappear between refreshes.

**Function/Class:** `fetchOrdersIfIdentifiersExist` / `fetchOrdersForAllOrganizations`

**Severity:** medium

**Confidence:** high — confirmed by adversarial verification

**Steps to reproduce:**

1. Open the Order Dashboard with no Customer filter.
2. In the Order Status filter tick **both** Complete and Cancelled, and pick a month/year.
3. Ensure the first unscoped search returns 500 rows so the fan-out triggers.
4. Make the reporting service fail intermittently for one status on one organization.
5. **Expected:** that organization's Complete orders still appear; only the Cancelled ones are missing.
6. **Actual:** that organization contributes nothing at all, and the response is still HTTP 200.

**Problem:** The new per-organization `catch` is coarser than the failure unit it guards. `fetchOrdersIfIdentifiersExist` builds one promise per *(identifier × status)* and awaits them with a bare `Promise.all`, which rejects on the first failure.

**Evidence:**

```ts
const promises = shouldUseReporting
  ? identifiers.flatMap((id) =>
      orderStatus.map((status) =>
        fetchReportingOrders({ ... searchStatus: status })
      )
    )
  : ...;

const results = await Promise.all(promises);   // line 70 — rejects on first failure
```

Reachability was verified end to end: `searchOrdersForTable.ts:62` passes `orderStatus: orderStatus || []`, `utils.ts:193` forwards it, and `utils.ts:113-116` passes `includeClosed` plus `searchMonth`/`searchYear`, so `shouldUseReporting` is true whenever a closed status is selected. Multi-status is real, not theoretical — `TableWithFilters/hooks.ts:319` passes `currentStatusFilter` from a multi-select, and `isFilteringClosedOrders` uses `intersection(CLOSED_JOB_STATUSES, currentStatusFilter)`.

**Impact:** Silent, non-deterministic partial data loss on the path this PR adds — the same failure class PP-2027 reports. **Not a regression:** pre-PR the fan-out did not exist, and the org-scoped branch failed harder (whole request 500). This is a new guard placed one level too high, not a behaviour that got worse.

**Fix:** Push the resilience down to the individual request, so a failing status costs only that status:

```typescript
const results = await Promise.all(
  promises.map((promise) =>
    promise.catch(() => [] as OrderFromSearch[])
  )
);
```

---

### 5. Swallowed failures never reach Sentry, and the client is never told the result is partial

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:119-134]**

> **In plain terms:** If one partner's order search fails every time it is tried, that partner's orders quietly disappear from the dashboard indefinitely. Nothing alerts anyone — it only shows up if someone happens to go looking through log warnings. The page reports success either way, so users have no way to know they are looking at an incomplete list.

**Function/Class:** `fetchOrdersForAllOrganizations`

**Severity:** medium

**Confidence:** high

**How to spot it:** Make one organization's search fail persistently. The dashboard returns HTTP 200 with that organization's orders absent; no Sentry issue is created and the response carries no partial-result marker.

**Problem:** The catch returns `[]` and emits only a `logger.warn`. It does not call `reportError` from `@proofed/shared/utils/throwSentryError` — the established helper in this repo, used by `handleEndpointError.ts:59` for every other failure in this route family. The same applies to the truncation branch at `utils.ts:119-124`.

**Evidence:** The catch block at `utils.ts:127-134` contains only `logger.warn(...)` and `return [] as OrderFromSearch[];`. There is no `reportError` import in the file. The success log at `searchOrdersForTable.ts:123-125` is `logger.info("Fetched filtered orders for table", { status: 200 })` — unchanged by this PR, and carrying no count of failed or truncated organizations.

**Impact:** Silent data loss with no alerting, and no measurement of how often the new fan-out partially fails. Given the fix is *about* orders going missing invisibly, reintroducing an invisible way for orders to go missing is worth closing.

**Fix:** Report to Sentry in the catch, and make partiality measurable in the success log:

```typescript
reportError(error, {
  operation: "order.search-by-organization",
  organizationId
});
```

Then add `failedOrganizationCount` and `truncatedOrganizationCount` to the `logger.info` at `searchOrdersForTable.ts:123`.

---

### 6. `orderStatus` has no length cap or de-duplication, and now multiplies against a server-supplied organization count

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/schema.ts]**

> **In plain terms:** The number of requests the server makes to other systems is the number of partners multiplied by the number of statuses ticked. The status list arrives from the browser and nothing limits its length or rejects repeats, so a single signed-in request can be crafted to make the server issue an enormous number of internal calls. Even with no bad actor, ordinary use now costs roughly a hundred times more internal traffic than before, on a page that refreshes itself every 30 seconds.

**Function/Class:** `searchOrdersForTableRequestSchema` / `fetchOrdersIfIdentifiersExist`

**Severity:** medium

**Confidence:** high — validated against the repo's installed yup

**Steps to reproduce:**

1. Authenticate to the creative portal.
2. POST to `/api/mixtures/orders/searchOrdersForTable` with `orgIds: []`, `orgGroupIds: []`, `searchMonth`/`searchYear` set, and `orderStatus` as a long repeated array, e.g. `Array(5000).fill("Canceled")`.
3. **Expected:** validation rejects the oversized/duplicated array.
4. **Actual:** it validates, and once the first unscoped search returns ≥500 rows the handler fans out to (organization count × 5000) upstream reporting calls.

**Problem:** No `.max()` and no uniqueness constraint on `orderStatus`; `oneOf` validates each item independently and is indifferent to repeats.

**Evidence:** The field as written:

```ts
orderStatus: Yup.array()
  .of(
    Yup.mixed<keyof typeof FilterableOrderStatuses>()
      .oneOf(validStatuses, "Invalid order status")
      .required("Order status is required")
  )
  .required(),
```

Run against yup 0.32.11 from this repo's `node_modules`: `{orgIds:[], orgGroupIds:[], orderStatus: Array(5000).fill("Canceled"), searchMonth:5, searchYear:2026}` → **PASS**, `orderStatus.length = 5000`. (`array().required()` in yup 0.32 does not enforce non-empty either, so `orgIds: []` passes.) The multiplication happens at `utils.ts:45-58` via `identifiers.flatMap((id) => orderStatus.map(...))` inside a bare `Promise.all` at `:70`, so `ORDER_SEARCH_CONCURRENCY = 10` throttles organizations but **not** the per-organization status fan-out — peak in-flight is `10 × orderStatus.length`.

**Impact:** Load amplification with a client-controlled multiplier. To be fair to the PR: **this is amplified, not introduced** — the identifier × status multiplication is unchanged from base, and a caller could already pad `orgIds`. What is new is that the multiplicand is now *server-supplied* (every organization), so no attacker-supplied ids are needed. It is also bounded — Next's default 1 MB body limit caps the array at roughly 90k entries — so this is a resource-exhaustion concern, not an unbounded one. No rate limiting exists in `withApiMiddleware`.

**Fix:** Constrain the input and flatten the concurrency pool:

```typescript
orderStatus: Yup.array()
  .of(/* unchanged */)
  .max(validStatuses.length, "Too many order statuses")
  .transform((statuses) =>
    Array.isArray(statuses) ? [...new Set(statuses)] : statuses
  )
  .required(),
```

Then route the inner `Promise.all` at `utils.ts:70` through `mapWithConcurrency` so the bound covers `organizations × statuses`, not just organizations.

---

### 7. The fan-out searches deactivated organizations

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:101]**

> **In plain terms:** The recovery step asks for every partner ever created — including ones that have been shut off and cannot have any live orders — and then makes a separate request for each. That is wasted work on every dashboard refresh for every user, and it grows permanently as more partners are added.

**Function/Class:** `fetchOrdersForAllOrganizations`

**Severity:** medium

**Confidence:** high

**How to spot it:** Not user-visible. Compare the call with the sibling caller: `api/mixtures/partnerProjects/utils.ts:17` passes `getOrganizations(requesterId, { status })`; this one passes `{}`.

**Problem:** `getOrganizations(requesterId, {})` applies no status scoping although the option exists and the returned type carries the flag.

**Evidence:** `GetOrganizationsOptions` exposes `status?: "A" | "D" | "B"` (`api/organizations/types.ts:24-28`) and `Organization` carries `active: boolean` (line 13). Neither is used at the call site. The only other production caller in `api/mixtures` does scope by status.

**Impact:** Every dormant, deleted or blocked organization costs one upstream search per poll, per user, indefinitely. At the author's measured 108 organizations, a plausible dormant fraction removes tens of calls per poll at zero behavioural cost. It may also pull in orders from deactivated organizations that the unscoped search would not have returned — worth confirming (see Open Questions).

**Fix:** `getOrganizations(requesterId, { status: "A" })`, or at minimum `organizations.filter(({ active }) => active)` before the map. Fold this into the try/catch from Issue 2.

---

### 8. The compiler is not checking the contract this PR changed

**[File: apps/creative-portal/api/orders/utils.ts:58-95]**

> **In plain terms:** The whole point of this half of the change is that a job lookup can now come back as "no job". The type system, which is supposed to guarantee that promise holds, has been measured and is not actually checking it. If someone later broke this in the obvious way, the build would stay green and only the tests would catch it.

**Function/Class:** `extractCurrentJobFromLiveStatusField` / `extractGranularCurrentJobFromLiveStatusField`

**Severity:** medium

**Confidence:** high — measured with the TypeScript 5.3.3 compiler API

**How to spot it:** Not user-reproducible. Inspect the inferred types.

**Problem:** Two independent gaps combine so that the declared return type is verified by nothing.

**Evidence:** Measured inferred types:

| expression | inferred type |
|---|---|
| `extractCurrentJobFromLiveStatusField` | `(liveStatus: string, options?: {...}) => string \| undefined` |
| `AI_JOB_FILTER_BY_PREFIX[x] ?? extractCurrentJobFromLiveStatusField(y)` | **`AiJobFilterValue`** |
| `extractGranularCurrentJobFromLiveStatusField` | `(liveStatus: string) => CurrentJobFilterValue \| undefined` — 0 diagnostics |

First, `extractCurrentJobFromLiveStatusField` has no explicit return type; its returns are `undefined`, `JobType`, and `options?.returnServiceList ? prefix : JobType.SERVICE`. Subtype reduction collapses `JobType | string` to `string`, erasing the enum. Second, `AI_JOB_FILTER_BY_PREFIX` is `Record<string, AiJobFilterValue>` (`utils.ts:44`) and `noUncheckedIndexedAccess` is not enabled (`packages/tsconfig/base.json` sets only `"strict": true`), so the index access is typed non-nullable — and `??` with a non-nullable left operand **discards the right operand's type entirely**.

**Impact:** Runtime behaviour is correct today, and the tests do cover it. But the one contract this PR changes is the one the compiler is not enforcing: if the function stopped returning `undefined`, or returned a raw service prefix on this path, nothing would fail to compile.

**Fix:** Both halves, verified together to give 0 diagnostics and to restore the correct union:

```typescript
// An index lookup can miss — say so, and `??` keeps the right operand's type.
const AI_JOB_FILTER_BY_PREFIX: Partial<Record<string, AiJobFilterValue>> = { ... };

export function extractCurrentJobFromLiveStatusField(
  liveStatus: string,
  options: { returnServiceList: true }
): string | undefined;
export function extractCurrentJobFromLiveStatusField(
  liveStatus: string,
  options?: { returnServiceList?: false }
): JobType | undefined;
```

`JOB_NAME_TO_TYPE` at `utils.ts:32` carries the same latent-`undefined` lie — pre-existing, same one-word fix.

---

### 9. The order-group fan-out has no per-item guard, and its input grew by two orders of magnitude

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/searchOrdersForTable.ts:96-108]**

> **In plain terms:** After the orders are filtered, the page makes one more request per order group. That step has no protection: if any single one fails, the whole page errors. That was low-risk when it ran over at most 500 orders; this change means it can now run over every live order in the system, so the same unprotected step is exercised far harder and takes far longer.

**Function/Class:** `searchOrdersForTable`

**Severity:** medium

**Confidence:** high

**How to spot it:** Not directly user-reproducible without an upstream failure. There is also **no `searchOrdersForTable.test.ts`** — the directory holds only `consts.ts`, `index.ts`, `schema.ts`, `searchOrdersForTable.ts`, `types.ts`, `utils.ts`, `utils.test.ts` — so this change has no coverage at all.

**Problem:** The rejection semantics are *unchanged* (base was `Promise.all`, which also fails fast — so this is not a behaviour regression). What changed materially is the exposure.

**Evidence:** `uniqueGroupIds` is derived from `filteredOrders` (`searchOrdersForTable.ts:78-86`), which derives from `orders`. Before: `orders` was one capped search, so `uniqueGroupIds.length ≤ 500`. After: `orders` is `uniqBy([...500 rows, ...N orgs × ≤500 rows])` (`utils.ts:198`), so the ceiling is now every live order in the system. `mapWithConcurrency`'s own doc (`packages/shared/utils/mapWithConcurrency.ts:6-11`) notes that on rejection *"other workers are not cancelled — they continue running and their writes to `results` are orphaned"*, so after the 500 is returned to the browser the pool keeps issuing discarded requests while the client retries in 30s.

**Impact:** Failure probability and duration both scale with a number that just grew ~100×. At concurrency 10 and ~200 ms per call, 2,000 group ids is roughly 40 s of serialised round trips on top of the fan-out.

**Fix:** Mirror the per-item catch already used in `fetchOrdersForAllOrganizations`, and add a handler test:

```typescript
const groupOrdersResults = await mapWithConcurrency(
  uniqueGroupIds,
  ORDER_SEARCH_CONCURRENCY,
  async (groupOrderGroupId) => {
    try {
      return await fetchOrders({ /* unchanged */ });
    } catch (error) {
      logger.warn("Failed to fetch group orders", { groupOrderGroupId });

      return [] as OrderFromSearch[];
    }
  }
);
```

Sibling mixture handlers do have handler-level tests (`api/mixtures/jobs/addNewJobs/addNewJobs.test.ts`, `api/mixtures/orders/getBulkActionsData/getBulkActionsData.test.ts`), so the pattern exists and was not followed here.

---

### 10. The concurrency bound and the Current Job sort change are both unpinned by tests

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:105 and apps/creative-portal/components/molecules/tables/TableWithFilters/utils.ts:107]**

> **In plain terms:** Two safeguards in this change can be removed without any test failing. First, the limit on how many partner requests run at once — take it away and the server would hammer the orders service with one simultaneous request per partner, and the suite stays green. Second, this change quietly moves orders that have not started work yet to a different position in the list; that reordering is deliberate and disclosed, but nothing verifies it.

**Function/Class:** `fetchOrdersForAllOrganizations` / `sortOrdersByJobs`

**Severity:** medium

**Confidence:** high — the concurrency gap proven by mutation

**How to spot it:** Not user-reproducible. Replace `ORDER_SEARCH_CONCURRENCY` at `utils.ts:105` with `organizations.length` and run the suite.

**Problem (a) — concurrency:** `utils.test.ts:126-142` asserts only `expect(mockFetchOrders).toHaveBeenCalledTimes(25)`, which is invariant under any concurrency limit. The mutation to `organizations.length` (fully unbounded) left **11/11 tests passing**.

**Problem (b) — sort:** `extractAndMap` (`TableWithFilters/utils.ts:88`) maps `undefined → -1`, and `REVERSE_ORDER_TO_JOBS` (`consts.tsx:53-59`) has `Service: 5`, so no-current-job rows move from rank 5 to −1. `JobType.SERVICE = "Service"` (`api/jobTypes/enums.ts:16`) confirms the pre-PR key really did hit the map. `grep -rn "sortOrdersByJobs" apps` returns only production files — zero test references, and `utils.test.ts:8` mocks `REVERSE_ORDER_TO_JOBS: {}` so ordering could not be asserted anyway.

**Evidence:** The sort chain at `hooks.ts:504-514` applies `.sort(sortOrdersByStatus).sort(sortOrdersByJobs).sort(sortByProjectName).sort(sortByPartnerName)`. Because `Array.prototype.sort` is stable and later sorts take priority, `sortOrdersByJobs` is a third-level tiebreaker — so the row moves to the front of **its partner+project block**, not to the top of the table. `areOrdersInSameGroup` (`utils.ts:152`) also now splits these rows out of the Service group for the deadline/creation tiebreak.

**Impact:** (a) the only thing preventing one simultaneous request per organization can be deleted with a green suite; (b) a disclosed-but-unverified UI reordering. The PR body does state *"Current Job sort | Sorted into the Service block | Sorts as 'no job' (-1)"*, so this is intended — it just is not held in place.

**Fix:** Assert peak in-flight count for (a), and add a `describe("sortOrdersByJobs")` using the real consts for (b):

```typescript
it("never runs more than ORDER_SEARCH_CONCURRENCY searches at once", async () => {
  let inFlight = 0;
  let peak = 0;

  mockGetOrganizations.mockResolvedValueOnce(
    Array.from({ length: 25 }, (_, index) => ({ id: index + 1 }))
  );
  mockFetchOrders.mockImplementation(async () => {
    inFlight += 1;
    peak = Math.max(peak, inFlight);
    await Promise.resolve();
    inFlight -= 1;

    return [];
  });

  await fetchOrdersForAllOrganizations(baseParams());

  expect(peak).toBeLessThanOrEqual(ORDER_SEARCH_CONCURRENCY);
});
```

---

### 11. The PR's claimed "Services column" fix does not correspond to the rendering path

**[File: apps/creative-portal/services/orders/utils.ts:59-76]**

> **In plain terms:** The PR says one of the things it fixes is the Services column printing the words "Current Job not ready". That column is actually drawn from a completely separate lookup and never used the code that was changed — so that particular symptom cannot have been caused by this code, and the test added for it covers a value nothing displays. Worth resolving before QA is asked to verify it, or QA will go looking for a symptom that was never there.

**Function/Class:** `extractServices` / `mapOrderFromApiToTable`

**Severity:** low

**Confidence:** high

**How to spot it:** Open `ServicesCell` and follow it to `ServicesCellContent`. Then grep for any read of the mapped row's `services` field in the dashboard.

**Problem:** The PR description's table claims *"Services column | Printed the literal `"Current Job not ready"` | Blank"*. The Order Dashboard's Services column does not read that value.

**Evidence:** `TableWithFilters/cells/ServicesCell/index.tsx:12-17` renders `<ServicesCellContent orderId={row.original.order.id} />`. `ServicesCellContent/index.tsx:27-55` fetches its own data via `useJobSearch` and derives the text from `serviceJob?.description`, not from `extractServices`. Grepping the dashboard for reads of `.services` on a mapped row returns nothing — only mocks and a skeleton placeholder.

**Impact:** No functional harm — the `extractServices` change is harmless and arguably more correct. But one of three claimed fixed surfaces is inert, and `services/orders/utils.test.ts:87-96` pins a field with no consumer. If the reported symptom was real, it came from somewhere else and is still unfixed.

**Fix:** Either correct the PR description and the manual-QA expectation, or identify the surface that actually rendered the literal string (an older table, an export, or the customer portal) and confirm whether it is still affected.

---

### 12. Truncation warning is arithmetically wrong on the reporting path

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/utils.ts:119-124]**

> **In plain terms:** The change adds a warning meant to flag partners that are too big for the system to fetch completely. On completed/cancelled searches that warning fires against a total added up across several separate requests, so it can cry wolf when nothing was actually cut off — and when something genuinely was, it reports a nonsensical number. This is the only operational signal the PR ships for its own known limitation.

**Function/Class:** `fetchOrdersForAllOrganizations`

**Severity:** low

**Confidence:** high

**How to spot it:** Not user-visible. Search logs for `"Order search truncated for organization"` and compare `returned` against the 500 cap it is measured against.

**Problem:** The cap check is applied to the flattened union of several independently-capped responses, not to any single response.

**Evidence:**

```ts
if (organizationOrders.length >= ORDER_SEARCH_ROW_CAP) {
  logger.warn("Order search truncated for organization", {
    organizationId,
    returned: organizationOrders.length
  });
}
```

`organizationOrders` is `results.flat()` across `orderStatus.length` reporting calls (`utils.ts:70-72`). With two statuses selected, 300 Complete + 250 Cancelled = 550 warns "truncated" though neither call hit its cap. Conversely three genuinely-capped calls report `returned: 1500` against a cap of 500.

**Impact:** False-positive alerts, and a misleading number in the true-positive case.

**Fix:** Do the cap check per underlying response inside `fetchOrdersIfIdentifiersExist`, or have it return `{ orders, truncated }`.

---

### 13. `getByName.test.ts`'s mock drops the service-header merge, so its exact-match assertions are false against reality

**[File: apps/creative-portal/api/organizations/getByName.test.ts:11-23, 45, 53, 65]**

> **In plain terms:** The new test for this helper checks the exact set of values sent with the request. But it stands in a simplified stub for a piece of shared plumbing, and that stub leaves out values the real plumbing always adds. So the test asserts something that could never be true in production, and it would not catch a name collision between the two sets of values.

**Function/Class:** `getOrganizations`

**Severity:** low

**Confidence:** high

**How to spot it:** Compare the mock at `getByName.test.ts:11-23` with the real `prepareServiceAxiosConfigs.ts:47-56`.

**Problem:** The mock returns `requestConfig?.options` verbatim; the real helper returns `[url, { ...options, headers: { ...serviceHeaders, ...options.headers } }]`. The three `toEqual({ requesterId: 1231 })` assertions are exact-match and would fail against the real merge, which always carries `api-key` and `Version`.

**Evidence:** To be clear about what *is* sound — the test is **not** vacuous. The mock is a pass-through of the object `getByName.ts:13-18` itself constructs, so it does observe production output, and reverting `getByName.ts:17` to `removeEmptyKeys(options)` makes the test fail with the real `Object.entries(undefined)` crash. The fix is genuinely pinned. The weakness is fidelity only.

**Impact:** The test cannot catch a header-key collision between `serviceHeaders` and a `GetOrganizationsOptions` key.

**Fix:** Have the mock reproduce the merge and relax the assertions:

```typescript
prepareServiceAxiosConfig: async (_serviceName, requestConfig) => [
  "http://organization-service",
  {
    ...requestConfig?.options,
    headers: {
      "api-key": "test-key",
      ...requestConfig?.options?.headers
    }
  }
]
```

then assert with `expect.objectContaining({ requesterId: REQUESTER_ID })`.

---

### 14. Convention and documentation nits

**[File: apps/creative-portal/api/mixtures/orders/searchOrdersForTable/consts.ts:1-2, utils.test.ts:43-44, api/organizations/getByName.ts:15-17]**

> **In plain terms:** A handful of small tidiness items — an undocumented magic number copied from another team's system, a type shortcut in a test that the rest of the codebase avoids, and a compiler suppression that now hides less than it used to. None of these affect what users see.

**Severity:** low

**Confidence:** high

**How to spot it:** Code health only — not user-reproducible.

**Problem and fix, item by item:**

**(a) Undocumented constants.** `ORDER_SEARCH_ROW_CAP = 500` mirrors a cap owned by another service, and the whole fan-out is gated on it — yet it carries no provenance. The repo has the exact analogue documented in one line: `api/jobs/search/consts.ts:1-2` — `/** Backend limit: max job IDs per single search request when searchBy=jobid */`. Add the same, naming which service and how 500 was determined. Consider homing it next to `fetchOrders`, since the cap is a property of every unscoped `fetchOrders` call, not of this endpoint.

**(b) `as any` in the test.** `logger: makeLogger() as any` with an eslint suppression, where five existing tests use `as unknown as Logger` and need no suppression (`fetchChargesForOrder.test.ts:239-243`, `submit/index.test.ts:64`, `ai-review-feedback.test.ts:18`, `verifyReviewerJobAssignment.test.ts:32`). Switching drops the disable comment.

**(c) The `@ts-ignore` in `getByName.ts` now hides less than it did.** It is still *required* — removing it yields `TS2345: Index signature for type 'string' is missing in type 'GetOrganizationsOptions'`, because that type is declared with `interface` (`api/organizations/types.ts:24`) and interfaces get no implicit index signature. But before this PR it masked *two* errors, the second being the real `undefined` crash. It now masks only the benign one, so deleting `?? {}` again would go unnoticed by the compiler. The clean form already exists in-repo at `api/utils/orders/fetchOrders.ts:14` — `...(options ? removeEmptyKeys(options) : {})`, no ignore, because `FetchOrdersArgs` is a `type` alias. Changing `GetOrganizationsOptions` from `interface` to `type` removes the need for the suppression entirely.

**(d) Stale test comment.** `utils.test.ts:144-157` documents an `Object.entries(undefined)` crash that cannot occur in that file (`getOrganizations` is mocked) and that the same PR fixed anyway. The assertion is not a no-op — dropping the second argument does fail it — but it now pins a call shape rather than a behaviour, and would fail a legitimate future tidy-up to `getOrganizations(requesterId)`. Either delete it (`getByName.test.ts:42-48` covers the real contract) or rewrite the comment.

**(e) Mixed lodash import styles** across the two files this PR edits — `import uniqBy from "lodash/uniqBy"` (`searchOrdersForTable.ts:1`) vs `import { uniqBy } from "lodash"` (`utils.ts:1`). Both pre-existing; the deep-import form dominates and avoids pulling all of lodash into the serverless bundle.

---

## Open Questions

Unconfirmed — please answer rather than treat as defects.

- **Does the orders service apply the `orgId` header as a filter *within* the requester's entitled set, or as an authoritative scope?** Every leg of the fan-out carries the same `requesterId` the previous single search carried, and the unfiltered organization list is already handed to the same browser by `GET /api/organizations` (`services/organizations/index.ts:45-53`, used at `TableWithFilters/hooks.ts:33`), so nothing in this repo indicates widened visibility. But the app layer applies no role check — `withApiMiddleware` defaults `requiredRoles: []` and `git grep requiredRoles apps/creative-portal/api` returns nothing — so the orders service is the sole authorization boundary. Concretely: for a requester entitled only to org 10, does a search with `searchBy: "orgId", searchValue: 42` return org 42's orders, or empty/403? — `api/mixtures/orders/searchOrdersForTable/utils.ts:108-117`
- **Is `ORDER_SEARCH_ROW_CAP = 500` a documented contract or an observation?** If the real cap is higher, `orders.length < 500` always short-circuits and the fix silently never fires. If it is ever lowered, the fan-out fires on every poll. Does the reporting service share the same cap? — `consts.ts:1`, `utils.ts:184`
- **Is 108 organizations the production count or the B2B-test count?** Every added organization is a permanent per-poll upstream call. — `utils.ts:101`
- **Should `getOrganizations(requesterId, {})` be scoped to `status: "A"`?** Beyond the wasted calls (Issue 7), could the fan-out surface orders from deactivated organizations that the unscoped search deliberately excluded? — `utils.ts:101`
- **Does the reporting service accept non-terminal statuses (`Active`, `In Queue`, `Overdue`) as `searchStatus`?** A mixed selection like `["Complete", "Active"]` issues one reporting call per organization for `Active`. If those are rejected, Issue 4's failure mode fires routinely rather than rarely. `Overdue` is a client-derived concept, not an OMS status. — `utils.ts:45-58`
- **Is `id` the same JavaScript type (number vs string) in the orders-service and reporting-service responses?** Both are cast to `OrderFromSearch[]` without validation, and `uniqBy([...], "id")` would not dedupe `1` against `"1"` — producing duplicate dashboard rows. — `utils.ts:198`
- **Should `extractCurrentJobFromLiveStatusField` use `startsWith` rather than `===`?** The constant is named `NO_CURRENT_JOB_PREFIX` but compared with strict equality. What is the full set of strings OMS emits — any variant wording, trailing punctuation, or double spacing would fall through to the `JobType.SERVICE` fallback and reintroduce exactly this bug, silently. — `api/orders/utils.ts:71`
- **Should a partial result be surfaced to the client?** Both the failed-organization path and the >500-per-organization truncation are currently log-only. Is a `truncated: true` flag and a UI banner wanted, or is silence the deliberate product call? — `utils.ts:119-134`
- **What is the platform request ceiling in front of this route?** No `vercel.json`, no `maxDuration`, and no axios timeout anywhere in the repo (`git grep` for `timeout|AbortController|axios.defaults|httpAgent` across `packages/shared/api` and `apps/creative-portal/api` returns nothing). This is pre-existing and systemic, not introduced here — but the PR turns one hung upstream call into a hung request holding a 10-wide worker pool, so the exposure grows ~110×. Worth a separate hardening ticket.

**Explicitly not raised as findings.** Two items were checked and judged to be documented scope boundaries rather than defects, because the PR states both in its own words: (1) an organization with >500 live orders still truncates — the per-organization search carries identical cap semantics, confirmed by reading `fetchOrders`/`fetchReportingOrders`; (2) orders with no current job cannot be filtered *for*. On (2), the "invisible" framing would be wrong in any case — `isAnyFilterPopulated` (`hooks.ts:202-217`) is satisfied by a status-only or customer-only filter, and `applyServerSideOrderFilters` skips the job block entirely when `currentJobType` is empty, so these orders stay visible.

A claim that `import { Logger } from "@logtail/next"` in `types.ts:1` risks pulling a server package into the client bundle was **investigated and refuted**: the same file on `develop` already relies on identical elision for `import { searchOrdersForTableRequestSchema } from "./schema"` (a value binding used only inside `InferType<typeof ...>`), and `verbatimModuleSyntax` / `importsNotUsedAsValues` / `consistent-type-imports` are all absent. `import type` would be tidier, nothing more.

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

**The validation suite was not run, and GitHub reports no CI checks on this branch.** The four gates CLAUDE.md treats as mandatory are entirely unverified for this PR. Re-run them before merging.

One partial data point: the four PR test files were executed in a throwaway worktree during review — **62 tests, 62 passed** (11 + 3 + 35 + 13), and `apps/creative-portal/vitest.config.ts:14` (`include: ["**/*.test.{ts,tsx}"]`, no `api/**` exclusion) confirms the new `api/**` test paths are genuinely collected, not silently skipped. That is not a substitute for the full suite, typecheck, lint or build.

---

## Tests

- ✅ Tests added for new code — 62 across 4 files, well above the bar for a bugfix
- ✅ Test *intent* is unusually good — fault tolerance is proven by mocking a rejection mid-fan-out rather than asserted, and the `getByName` test was verified to fail against the unfixed code
- ✅ Vitest genuinely collects the new `api/**` paths (verified, not assumed)
- ✅ The `NO_CURRENT_JOB_PREFIX` half is properly pinned — deleting the early return fails 5 tests across 2 files
- ✅ Cap boundary, union-not-replace, `uniqBy` dedup, per-organization catch, `searchBy: orgId` and the truncation threshold all fail under mutation
- ❌ **Closed-order / reporting wiring is unpinned** — stripping all four params leaves 11/11 passing (Issue 3)
- ❌ **`includeClosed: hasClosedOrderStatus` is unpinned** — forcing `true` leaves 11/11 passing (Issue 3)
- ❌ **`ORDER_SEARCH_CONCURRENCY` is unpinned** — replacing it with `organizations.length` leaves 11/11 passing (Issue 10)
- ❌ **`getOrganizations` rejecting is untested** — no `mockRejectedValue` on it anywhere (Issue 2)
- ❌ **No `searchOrdersForTable.test.ts`** — the handler's `mapWithConcurrency` change has zero coverage (Issue 9)
- ❌ **`sortOrdersByJobs` has no test** — zero references in any test file (Issue 10)
- ⚠️ `getByName.test.ts` mock drops the service-header merge (Issue 13)
- ⚠️ `utils.test.ts:90-103` — a 501-element `arrayContaining` costs 536 ms of a 628 ms file. Prefer `toHaveLength(ORDER_SEARCH_ROW_CAP + 1)` plus `expect(orders.slice(0, ORDER_SEARCH_ROW_CAP)).toEqual(cappedPage)`
- ⚠️ Pre-existing branches still untested: empty-identifiers early return, `orderGroupId`, `batchIdentifier`, and the `orgIds`/`orgGroupIds` branch — cheap to add now the scaffolding exists

### Suggested manual QA script

1. **Ticket verification (still unticked in the PR).** On B2B test, open order 21341, note its current job is a Review job. Go to the Order Dashboard, apply Current Job = Review with no Customer filter. **Expect:** order 21341 appears. This is the PR's own outstanding manual check.
2. Repeat step 1 for Service, QA, Return, AI Pre-edit and AI Post-edit against orders you have confirmed from the side panel. **Expect:** each returns its matching orders.
3. Find an order whose status reads "Current Job not ready". With **no** Current Job filter applied, confirm it still appears in the list. Then tick **all six** Current Job options and confirm it does not. **Expect:** exactly that — this is intended, disclosed behaviour, not a bug.
4. For the same order, check where it now sits in the list relative to others from the same partner and project. **Expect:** it sorts ahead of Return/Review/QA/AI/Service rows within its partner+project block. Verifies Issue 10(b).
5. Check the Services column for that order. **Expect:** blank. If it shows literal text, the surface named in the PR description is elsewhere — see Issue 11.
6. Select Order Status = Complete **and** Cancelled together with a past month and no Customer filter. **Expect:** results for that month. Verifies the reporting path that Issue 3 shows is untested. Note PP-2083 covers a related pre-existing gap here.
7. Time a dashboard load with no Customer filter, then leave the tab open for two minutes. **Expect:** it stays responsive across auto-refreshes. Watch for latency growth — relevant to Issues 6 and 9.
8. Switch to another tab and back several times. The query uses `refetchOnWindowFocus: "always"`, so each switch triggers a full fan-out. **Expect:** no visible degradation.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ✅ Core fix is sound and the re-diagnosis is well-evidenced |
| Regression risk | ⚠️ Medium — new hard dependency on the organization service; no cross-app or caller breakage found |
| Tests | ⚠️ Good volume and intent, but three mutation-proven gaps on the riskiest paths |
| Accessibility | n/a — no UI change |
| Error handling | ❌ Unguarded `getOrganizations`; coarse catch granularity; no Sentry on new failure paths |
| Security | ❌ Service credential written to logs (Issue 1). `/security` still required per CLAUDE.md |
| Code quality | ✅ Conventions, reuse, import/interface ordering and Prettier all verified clean |
| Validation suite | ⏭️ Not run — user opted out; **and no CI checks exist on this branch** |
| Mergeable state | ⚠️ GitHub reports `clean`, but nothing has been validated |

---

## Recommendation

**Request changes.**

The investigation behind this PR is genuinely strong — the author disproved the ticket's stated cause with instrumented evidence, found the real one, measured the cost of the fix, and documented the trade-offs honestly. The `NO_CURRENT_JOB_PREFIX` half is clean, minimal and correctly placed at the single shared entry point. Reuse discipline is good throughout. Most of what follows is about the resilience and observability of the fan-out, not the diagnosis or the approach.

**Blocking:**

1. **Stop logging the raw error object** (Issue 1) — it writes the orders-service `api-key` into Logtail on every upstream failure, at fan-out frequency. Log the message and status instead.

**Before merge:**

2. Wrap `getOrganizations` and fall back to the capped list (Issue 2) — the PR's own design rule, applied to the one call that skipped it. Add `status: "A"` while there (Issue 7).
3. Add the closed-order/reporting test through `fetchFilteredOrders` (Issue 3) — mutation proved all four params can be stripped with a green suite.
4. **Run the validation suite.** `test`, `typecheck`, `lint`, `build` have not been run here and no CI ran on the branch. CLAUDE.md treats any failure as a hard blocker, and right now the state is simply unknown.
5. Tick the outstanding manual check — confirm order 21341 under the Review filter on B2B test.
6. Run `/security` — the credential finding is exactly the class of issue that pass is for.

**Worth doing in this PR:**

7. Per-request catch inside `fetchOrdersIfIdentifiersExist` (Issue 4) and per-item catch on the group fan-out (Issue 9).
8. `reportError` on the swallowed failure, plus failed/truncated counts in the success log (Issue 5).
9. `.max()` + de-duplication on `orderStatus` (Issue 6).
10. Concurrency-bound and `sortOrdersByJobs` tests (Issue 10).

**Follow-ups, not blocking:** explicit return types on the job extractors (Issue 8); resolve the Services-column claim (Issue 11); truncation-warning arithmetic (Issue 12); test-mock fidelity (Issue 13); the constant documentation and `@ts-ignore` tidy-up (Issue 14). The absence of HTTP timeouts across the codebase, and the `getOrderActivityLog` instance of the same raw-error logging, both deserve their own tickets.
