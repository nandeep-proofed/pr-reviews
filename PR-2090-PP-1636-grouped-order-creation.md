# PR Review: feature/PP-1636: Enhance order creation process to support order grouping

**PR:** https://github.com/Proofed/B2BWebserver/pull/2090
**Jira:** https://proofed.atlassian.net/browse/PP-1636
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Display "Group related orders" when org group has Allow Order Grouping enabled AND >1 orders | `shouldShowGroupingOption` checks `allowOrderGrouping && documents.length > 1` in `DeliverySection/index.tsx` | ✅ Addressed |
| Hide option when conditions not met | `useEffect` resets `shouldGroupOrders` to `false` when `shouldShowGroupingOption` is false | ✅ Addressed |
| Information icon with tooltip text | Tooltip component with correct text: "Check this if your orders belong together..." | ✅ Addressed |
| First order sends OrderGroupId: "new", captures returned grouping ID | `effectiveOrderGroupId = activeOrderGroupId \|\| "new"`, then `activeOrderGroupId = response.orderGroupId` | ✅ Addressed |
| Subsequent orders use same grouping ID | Sequential loop with `activeOrderGroupId` propagated to all subsequent orders | ✅ Addressed |
| Ungrouped orders created without grouping | `effectiveOrderGroupId` is `undefined` when `shouldGroupOrders` is false | ✅ Addressed |
| Deadline uses sum of word counts across all grouped orders | `groupTotalWordCount` computed as `reduce(sum + doc.length)` on frontend; `wordCountForDeadline = groupTotalWordCount ?? document.length` on backend | ✅ Addressed |
| Stored word count per order remains unchanged | `groupTotalWordCount` only used in `calculateDeadline`, not in order body | ✅ Addressed |
| All orders in group assigned same deadline | Each order calculates deadline using same `groupTotalWordCount`, yielding approximately the same result | ⚠️ Partial |
| All jobs within grouped orders have same return time | `jobReturnTime = orderGroupId ? deadlineWithOrderedJobs.deadline : undefined` applied to all jobs | ✅ Addressed |
| OrderGroupId sent as header to backend service | `addOrder` and `addOrderStreamedJson` both conditionally add `OrderGroupId` header | ✅ Addressed |
| Jira specifies "radio button" | Implementation uses checkbox (`FormikCheckboxSingle`) | ⚠️ Partial (functionally equivalent) |

---

## Architecture Analysis

The PR adds order grouping support across the full stack within the customer portal:

- **Frontend (create-orders page):** Sequential order creation loop enhanced with `activeOrderGroupId` tracking. First order sends `"new"`, subsequent orders reuse the returned group ID. Total word count is pre-computed and passed along.
- **Backend API layer:** Both the standard and streaming endpoints accept `orderGroupId` and `groupTotalWordCount` via body/form fields, then forward `OrderGroupId` as a header to the downstream Orders service.
- **Deadline calculation:** `calculateDeadline` now accepts `groupTotalWordCount` and uses it (when present) instead of per-document word count.
- **Job return times:** Grouped orders assign all jobs the same return time (the order-level deadline) instead of incremental accumulated durations.
- **Shared packages:** `StreamUploadConfig.parseFormFields` signature updated to accept optional headers; `createStreamingService` updated to strip/forward `headers` from input.

The approach is sound and follows existing patterns. The grouping identifier flows correctly through the system.

---

## Issues Found

### 1. Redundant `orderGroupId` parameter in `addOrder.ts`

**[File: apps/customer-portal/api/utils/orders/addOrder.ts]**
**Function/Class:** addOrder
**Severity:** low
**Problem:** The function signature includes `orderGroupId?: string` as a separate destructured parameter (line 18), but the code actually reads `body.orderGroupId` (line 27) instead. The standalone parameter is never used.
**Impact:** Dead code that could mislead future developers into thinking it's used independently.
**Fix:** Remove the unused `orderGroupId` parameter from the function signature:

```typescript
export const addOrder = async ({
  deliveryConfigurationID,
  requesterId,
  body
}: {
  deliveryConfigurationID: number;
  requesterId: number;
  body: CreateOrderBE;
}) => {
```

### 2. Redundant payload construction in `createOrder.ts` endpoint

**[File: apps/customer-portal/api/orders/createOrder/createOrder.ts]**
**Function/Class:** createOrderEndpoint
**Severity:** low
**Problem:** `orderGroupId` and `groupTotalWordCount` are destructured from `body` (line 25) and then spread back into `body` to create `createOrderPayload` (lines 27-31). Since they're already part of `body`, the destructuring and re-assignment are redundant — `const createOrderPayload = body` would suffice.
**Impact:** No functional impact, but adds noise and suggests the fields need special handling when they don't.
**Fix:** Simplify to:

```typescript
const createOrderPayload = body;
```

### 3. Redundant `orderGroupId` extraction in return values

**[File: apps/customer-portal/api/utils/orders/addOrder.ts]**
**Function/Class:** addOrder (return statement)
**Severity:** low
**Problem:** `return { ...data, orderGroupId: data?.orderGroupId }` — `orderGroupId` is already included in `...data`, making the explicit property redundant. Same issue in `addOrderStreamedJson` in `createOrderAndJobs.ts` (line 66).
**Impact:** No functional impact, but obscures intent.
**Fix:** Either return `data` directly, or if the intent is to guarantee the property exists even when absent:

```typescript
return data;
```

### 4. Unsafe type assertions in `stream.ts`

**[File: apps/customer-portal/api/orders/createOrder/stream.ts]**
**Function/Class:** businessLogic
**Severity:** medium
**Problem:** Multiple `as` type casts are used for cached results (line 115: `cached.result as { id: number; orderGroupId?: string }`) and in-flight results (lines 123-126). These bypass TypeScript's type system rather than properly typing the cache/inflight data structures.
**Impact:** If the cache or in-flight promise resolves to a different shape, TypeScript won't catch the mismatch. This could cause runtime errors that are hard to debug.
**Fix:** Type the `idempotencyCache` and `inflightOrders` maps generically, or create a named type for the business logic result:

```typescript
type OrderResult = { id: number; orderGroupId?: string };

// Then type the cache/inflight structures appropriately
```

### 5. Inconsistent response shape in `formatResponse`

**[File: apps/customer-portal/api/orders/createOrder/stream.ts]**
**Function/Class:** formatResponse
**Severity:** medium
**Problem:** `results: order.results ?? order` creates two possible response shapes. If `order` has a `results` property, that's used; otherwise the entire `order` object becomes `results`. This dual-path fallback is confusing and could produce inconsistent API responses.
**Impact:** Frontend consumers might receive different `results` shapes depending on the code path. The non-stream endpoint always returns `results: order`, so there's a potential inconsistency between the two endpoints.
**Fix:** Determine the canonical response shape and use it consistently. If `businessLogic` always returns the order object directly (not wrapped in `results`), simplify to:

```typescript
const formatResponse = (order: {
  id: number;
  orderGroupId?: string;
}) => ({
  message: "Order created successfully",
  results: order,
  orderGroupId: order.orderGroupId
});
```

### 6. Grouped deadlines may differ slightly between orders

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/helpers/calculateDeadline.ts]**
**Function/Class:** calculateDeadline
**Severity:** medium
**Problem:** The Jira requires "All orders in the group must be assigned the same deadline." However, each order's deadline is calculated independently via separate API calls to `getDeadlineWithOrderedJobs`. Since these calls happen sequentially over time, business hours calculations and rounding could produce slightly different deadlines for different orders in the same group, especially near business-day boundaries.
**Impact:** Orders in the same group could have different deadlines by minutes or even a full business day if creation straddles a boundary.
**Fix:** Consider calculating the deadline once for the first order and reusing it for subsequent orders in the group, similar to how `activeOrderGroupId` is tracked on the frontend. Alternatively, the backend Orders service may handle this via the group ID — verify with the API team.

### 7. Content-Type header removal in streaming service

**[File: packages/shared/services/streamFile/index.ts]**
**Function/Class:** createStreamingService
**Severity:** low
**Problem:** The explicit `"Content-Type": "multipart/form-data"` was removed and replaced with `...headers`. While axios does auto-set Content-Type with proper boundary for FormData, the change means if `headers` is `undefined`, the headers object becomes `{}` (empty). This works because axios detects FormData, but it's an implicit behavior dependency.
**Impact:** Low — axios handles FormData Content-Type correctly. However, if a caller passes a custom `Content-Type` in `headers`, it could override the multipart boundary, breaking file uploads.
**Fix:** Consider explicitly preserving multipart detection:

```typescript
headers: {
  // axios auto-sets Content-Type with boundary for FormData
  ...headers
}
```

The current code already has the comment. Just ensure no callers pass `Content-Type` in the `headers` object.

### 8. `useEffect` for resetting `shouldGroupOrders` may cause unnecessary renders

**[File: apps/customer-portal/components/organisms/OrderCreation/partials/DeliverySection/index.tsx]**
**Function/Class:** DeliverySection
**Severity:** low
**Problem:** The `useEffect` that resets `shouldGroupOrders` to `false` when `shouldShowGroupingOption` becomes false runs `formikProps.setFieldValue` on every render where the condition is false — not just on transitions from true to false. This means every render with 0 or 1 documents triggers a Formik field update.
**Impact:** Minor performance concern — unnecessary Formik re-renders when the field is already `false`.
**Fix:** Only reset when transitioning from visible to hidden:

```typescript
const prevShouldShow = useRef(shouldShowGroupingOption);

useEffect(() => {
  if (prevShouldShow.current && !shouldShowGroupingOption) {
    formikProps.setFieldValue("shouldGroupOrders", false);
  }
  prevShouldShow.current = shouldShowGroupingOption;
}, [shouldShowGroupingOption]);
```

Or guard with a value check:

```typescript
useEffect(() => {
  if (!shouldShowGroupingOption && formikProps.values.shouldGroupOrders) {
    formikProps.setFieldValue("shouldGroupOrders", false);
  }
}, [shouldShowGroupingOption]);
```

---

## Tests

- ❌ No unit tests added for `calculateDeadline` changes (groupTotalWordCount logic)
- ❌ No unit tests added for `createOrderAndJobs` changes (orderGroupId propagation, job return time logic)
- ❌ No unit tests added for `addOrder` / `addOrderStreamedJson` header logic
- ❌ No unit tests added for `DeliverySection` grouping checkbox display/hide logic
- ❌ No unit tests added for `create-orders` page grouped submission flow
- ❌ No unit tests added for `parseFormFields` new fields parsing
- ❌ No existing test files found for any of the modified modules
- ⚠️ Project requirement states "Every PR must include tests for new code"

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Mostly correct — deadline consistency across grouped orders is a concern |
| Regression risk | ⚠️ Medium — shared package changes (`streamFile`, `fileUploadStream`, `streamUpload` types) affect all streaming consumers |
| Tests | ❌ No tests added |
| Code quality | ⚠️ Redundant code, unsafe type assertions, inconsistent response shapes |
| Mergeable state | ❌ Dirty |

---

## Recommendation

**Request changes**

1. **Add unit tests** — at minimum for `calculateDeadline` (groupTotalWordCount), `createOrderAndJobs` (orderGroupId flow, job return time), and `DeliverySection` (show/hide logic). This is a project requirement.
2. **Fix the grouped deadline consistency issue** — either calculate once and reuse, or confirm the backend Orders service handles this via the group ID.
3. **Remove redundant `orderGroupId` parameter** from `addOrder.ts` function signature (unused dead code).
4. **Clean up redundant return statements** in `addOrder.ts` and `addOrderStreamedJson` where `orderGroupId` is explicitly re-added from the spread object.
5. **Address unsafe type casts** in `stream.ts` — properly type the cache and in-flight maps.
6. **Clarify `formatResponse`** — the `order.results ?? order` fallback creates ambiguous response shapes.
7. **Guard the `useEffect` reset** of `shouldGroupOrders` to avoid unnecessary Formik updates.
8. **Confirm with design** whether checkbox vs. radio button (per Jira spec) is intentional.
