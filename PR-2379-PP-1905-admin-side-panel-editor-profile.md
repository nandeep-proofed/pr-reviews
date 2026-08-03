# PR Review: feature/PP-1905: Open admin side panel from editor profile job links

**PR:** https://github.com/Proofed/B2BWebserver/pull/2379
**Jira:** https://proofed.atlassian.net/browse/PP-1905
**Status:** Code Review
**Branch:** `feature/PP-1905-admin-side-panel-editor-profile` → `develop`
**Size:** 18 files, +723 / −209

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Clicking any job in the assigned jobs list opens the admin side panel | `JobsTab.onRowClick` looks the clicked job up in `flatAssignedRows`/`flatAvailableRows` and pushes `?orderId=<job.orderId>&jobId=<job.id>`; `MemberOrderSidebar` (new page partial) renders `OrderSidebarProvider` + `OrderManagementSidebar` — the same panel as the order management dashboard | ✅ Addressed |
| 1. Clicking any job in job history opens the admin side panel | `HistoryTab.onRowClick` pushes `?orderId=<item.original.order_id>&jobId=<item.original.id>`; `order_id` is a required field on `JobHistoryRowItem` and is populated in `user-job-history/utils.ts` (`order_id: job.orderId.toString()`) | ✅ Addressed |
| 1a. The user/client side panel must not open from this context | `JobSidebar` import and render removed from both `JobsTab` and `HistoryTab`; a repo-wide grep under `components/pages/team-members` shows no remaining reference outside test mocks. `CalendarTab` and `SettingsTab` have no job links | ✅ Addressed |
| 2. Panel must default to the job card for the job that was clicked | New optional `initialJobId` on `OrderSidebarProviderProps` → context → `OrderJobs` resolves it to `initialJobIndex` and uses it for both `initialSlide` and the `slideTo` effect | ✅ Addressed |
| 2a. Order management dashboard must keep defaulting to the current job | `components/pages/admin-area/orders/index.tsx` does not pass `initialJobId`, so `targetJobIndex` falls back to `currentJobIndex` — unchanged | ✅ Addressed |
| Testing note 6 — clicking a profile job while the client panel is open must open the admin panel | Unreachable by construction: the client panel can no longer be rendered on the profile at all | ✅ Addressed |
| Testing note 7 — client panel must not be openable from any job link on the profile | Covered by the removals above; asserted by `JobsTab.test.tsx` and `HistoryTab.test.tsx` ("never renders the user/client JobSidebar") | ✅ Addressed |

**Changes beyond the Jira scope:**

- `services/jobs/index.ts` and `services/customers/index.tsx` — relative `"api/jobs"` / `"api/customers"` URLs made absolute. **In practice required** for the feature: `resolveHref`-independent axios relative resolution means `api/jobs` resolves against the current directory, so it worked from `/orders` (→ `/api/jobs`) but 404s from `/team/profile/123` (→ `/team/profile/api/jobs`). I confirmed `apiRoutes.jobs === "/api/jobs"` and that no other relative axios URL remains anywhere in `creative-portal` or `packages/shared`.
- `useOrderSidePanelData` `onError` — `router.replace("")` → explicit `{ pathname, query }`. Also effectively required; see Architecture Analysis for verification of the "throws on dynamic routes" claim.
- `packages/shared/utils/formatWordQuantity.ts` — locale pinned to `en-US`. **Genuine scope creep**; see Issue 5.
- `OrderManagementSidebar.setBatchName` widened to optional, and `useOrderMoreOptionsDropdownList` now hides "Show group" when no handler is supplied. Both are consequences of reusing the panel outside the dashboard — in scope, but untested (Issue 6).

---

## Architecture Analysis

The approach is the right one and matches Orlin's guidance on the ticket ("we would need to create a new component"): rather than teaching `JobSidebar` to behave like the admin panel, the PR reuses the *existing* admin panel (`OrderSidebarProvider` + `OrderManagementSidebar`) behind a thin new page partial, and adds one optional input (`initialJobId`) to select the opening card. Panel state lives in the URL, matching the dashboard's `?orderId=` convention.

Things I verified rather than assumed:

- **The `router.replace("")` fix is real.** In Next 14, `resolveHref` builds its base from `router.pathname`, not `router.asPath` (`node_modules/next/dist/client/resolve-href.js:40`). So on `/orders` (pathname `/`) `""` resolves to `/` and correctly clears the query — which is why the dashboard's existing close works. On `/team/profile/[memberId]` it resolves to the literal un-interpolated pattern, and `router.change` then throws "missing query values … to be interpolated properly". The replacement preserves `memberId` and every other route/query param, and still produces `/` on the dashboard, so there is no regression there.
- **No privilege escalation.** The profile page is gated on `[Admin, Superadmin, ServiceDelivery, ServiceSupport]`; the orders dashboard on those plus `Returner`. The profile's role set is a strict subset, so no role gains access to order-management actions it couldn't already reach via `/orders`.
- **Click-outside doesn't fight the row click.** `useOrderManagementSidebar`'s document listener skips targets with a `TR` ancestor (`hasAllowedParentTag(..., ["TR"], ["THEAD"])`), and both `JobTable` and `HistoryTable` render real `<table>/<tr>` markup via react-table. So clicking a second job row while the panel is open switches orders instead of closing the panel — same as the dashboard.
- **The always-mounted provider doesn't fetch order data when closed.** `useOrderByIdQuery`, `useOrderSupportDocumentsQuery`, `useOrderJobTaskQuery` and `useCustomerQuery` are all gated on `deferredOrderId`, and `useJobsQuery` on `!!orderId`. (One ungated query does leak through a different path — Issue 3.)
- **Empty "more options" dropdown is not a risk.** `OrderSidebarHeader` guards with `!!moreDropdownItems?.length`, so a cancelled ungrouped order on the profile renders no dropdown rather than an empty one.

`MemberOrderSidebar` follows the repo's `index.tsx`-is-UI-only convention (state and callbacks in a sibling `hooks.ts`). It has no `types.ts`, which is fine — it takes no props, matching `CalendarTab`.

---

## Issues Found

### 1. `initialJobIndex` and `currentJobIndex` are indices into two differently-ordered arrays

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/index.tsx]**

**Function/Class:** OrderJobs

**Severity:** medium

**Problem:** `targetJobIndex = initialJobIndex ?? currentJobIndex` combines two indices that are not measured against the same array. `initialJobIndex` (new) is a `findIndex` over `sortedJobs` — `jobs` sorted by `order.jobSequence`. `currentJobIndex` comes from the provider and is a `findIndex` over the raw react-query `jobs` array, memoised on `[order, jobs]`. `OrderJobs` sorts in place (`jobs.sort(...)` mutates), so the array identity never changes and the provider's memo never recomputes against the sorted order. The slides, meanwhile, are rendered from `sortedJobs`. The whole reason `sortJobsByJobSequence` exists is that the API does not guarantee `jobSequence` order, so the two can genuinely disagree.

**Impact:** On the fallback path — which is every order-management-dashboard open, plus every profile open where `jobId` isn't in the order (group navigation, a stale link) — the panel can land on the wrong job card whenever the jobs endpoint returns a different order than `jobSequence`. This is pre-existing behaviour rather than a regression, but the PR now places both index spaces in one expression, and the new test encodes the assumption that they agree (`jobs: [101, 102, 103]` with `currentJobIndex: 2`), so the mismatch is invisible to the suite.

**Fix:** Use the already-correct sorted-array index as the fallback. `initialSlideIndex` in this same file computes exactly that (index of `order.currentJobId` in `sortedJobs`, defaulting to 0), which makes the third expression redundant at the same time:

```tsx
// Card the panel opens on: the explicitly requested job (e.g. a
// job link on the editor profile) wins over the order's current job.
// Both indices are resolved against sortedJobs — the array the
// slides are rendered from.
const targetJobIndex = initialJobIndex ?? initialSlideIndex;
```

then `initialSlide={targetJobIndex}` and drop the `?? initialSlideIndex` tail. Worth adding a test case where the mocked `jobs` are returned out of `jobSequence` order to lock the behaviour in. If you'd rather not touch the dashboard's fallback in this PR, at minimum add a comment recording that `currentJobIndex` is measured against the unsorted array, and raise a follow-up.

### 2. Panel URLs are built by string concatenation and drop every other query param

**[File: apps/creative-portal/components/pages/team-members/profile/[memberId]/partials/MemberOrderSidebar/hooks.ts]**

**Function/Class:** useMemberOrderSidebar — `setActiveOrderId`

**Severity:** low

**Problem:** `setActiveOrderId` (and both `onRowClick` handlers in `JobsTab`/`HistoryTab`) rebuild the URL as `` `${appRoutes.teamMembersProfile(userId)}?orderId=…` ``, discarding anything else that was on the query string. The sibling fix in this same PR — `useOrderSidePanelData`'s `onError` — deliberately does the opposite, spreading `router.query` and deleting only `orderId`/`jobId`.

**Impact:** No user-visible breakage today; the profile route carries no other query params. It becomes a real bug the moment one is added (a tab key, a filter, a `?new` flag), and the inconsistency inside a single PR makes the safer pattern easy to miss later.

**Fix:** Use the object form consistently, as the `onError` handler now does:

```ts
const setActiveOrderId = useCallback(
  (newOrderId: string | undefined) => {
    if (!userId) return;

    const query = { ...router.query };

    delete query.jobId;

    if (!newOrderId) {
      delete query.orderId;
      router.replace({ pathname: router.pathname, query }, undefined, {
        scroll: false
      });

      return;
    }

    router.push(
      { pathname: router.pathname, query: { ...query, orderId: newOrderId } },
      undefined,
      { scroll: false }
    );
  },
  [router, userId]
);
```

The existing unit tests assert the concatenated string form, so they'd need updating alongside.

### 3. The order panel now mounts on every profile page load, firing one ungated query

**[File: apps/creative-portal/components/pages/team-members/profile/[memberId]/index.tsx]**

**Function/Class:** TeamProfilePage

**Severity:** low

**Problem:** `<MemberOrderSidebar />` renders unconditionally, so `OrderSidebarProvider` → `OrderManagementSidebar` → `OrderManagementModals` → `OrderEditDetailsModal` mounts on every profile visit even with no `orderId`. `OrderEditDetailsModal` calls `useLanguageLocalizationOptions()`, which issues `useMasterReferenceQuery(ReferenceName.LanguageLocalization)` with no `enabled` guard (`packages/shared/hooks/useLanguageLocalizationOptions.ts`). Everything genuinely order-scoped is correctly gated on `deferredOrderId` / `!!orderId`, so this is the one that leaks.

**Impact:** One extra master-reference request per profile page load (react-query caches it for the session), plus the mount cost of the whole modal tree on a page that mostly won't open an order. Small, but it's a cost the profile page didn't previously pay.

**Fix:** Gate the mount on the query param:

```tsx
{router.query.orderId !== undefined && <MemberOrderSidebar />}
```

Trade-off to weigh: conditional mounting unmounts `OrderSidebar` the instant `orderId` clears, so the panel's close transition is skipped. If that animation matters, the cleaner fix belongs upstream — an `enabled: isOpen`-style guard inside `OrderEditDetailsModal`'s hooks, which would also benefit the dashboard.

### 4. A shared `?orderId=&jobId=` link lands on the Settings tab

**[File: apps/creative-portal/components/pages/team-members/profile/[memberId]/index.tsx]**

**Function/Class:** TeamProfilePage / Tabs

**Severity:** low

**Problem:** `useTabs` keeps the active tab in local React state with no URL sync, and `items[0]` is Settings. Within a session the tab survives (a same-route `router.push` re-renders rather than remounts), so clicking a job never bounces the admin out of Jobs/History — good. But on a refresh or on a link opened by another admin, the panel opens on top of the Settings tab.

**Impact:** The shareability and refresh-survival benefits the ticket comment calls out are only partly delivered: the panel is right, the page behind it isn't. Cosmetic, not broken.

**Fix:** Either sync the tab key into the URL (`?tab=history`) and pass `selectedTabKey`/`onChange` to `Tabs` — `useTabs` already supports controlled mode — or, if that's out of scope, drop the "shareable" framing from the PR description and note the limitation so QA doesn't file it.

### 5. `formatWordQuantity` locale pinning is unrelated scope and changes rendering for all users

**[File: packages/shared/utils/formatWordQuantity.ts]**

**Function/Class:** formatWordQuantity

**Severity:** low

**Problem:** The change is motivated in the PR description as a test fix ("its unit test failed on en-IN systems"), but `new Intl.NumberFormat()` reads the *browser's* locale at runtime, not just the test host's. Pinning to `en-US` changes what real users see. It also has nothing to do with PP-1905, sits in `packages/shared`, and therefore ships to the customer portal as well.

**Impact:** A user on an `en-IN` browser previously saw `10,00,000 words` and will now see `1,000,000 words`. That is arguably the *correct* outcome — it matches `formatPriceWithCurrencySymbol` and `formatCurrency`, both of which already pin `en-US` (I confirmed both exist, so the code comment is accurate) — but it's a product decision being made inside an unrelated ticket, with no test asserting the new behaviour.

**Fix:** Preferably split into its own commit/ticket so it's visible in the release notes. If it stays here, add a regression test that pins the intent rather than relying on the host locale:

```ts
it("uses en-US digit grouping regardless of host locale", () => {
  expect(formatWordQuantity(1000000)).toBe("1,000,000");
});
```

### 6. Three behaviour changes outside the profile ship without tests

**[File: apps/creative-portal/components/pages/admin-area/orders/hooks.tsx]**

**Function/Class:** useOrderSidePanelData (plus `OrderManagementSidebar` and `useOrderMoreOptionsDropdownList`)

**Severity:** low

**Problem:** The 23 new tests cover the profile-side additions well, but the three edits to shared admin-panel code have none:

- `useOrderSidePanelData`'s `onError` param-stripping — the change that actually prevents a thrown error, and the one most likely to regress if someone "simplifies" it back to `router.replace("")`.
- `OrderManagementSidebar`'s new `setBatchName`-optional branch — no test that the batch name renders as plain text (not a `RawButton`) when the handler is absent, nor that the dashboard still gets the link.
- `useOrderMoreOptionsDropdownList` hiding "Show group" when `onOrderGroupClick` is absent while still showing it on the dashboard.

**Impact:** CLAUDE.md requires tests for new code. These are exactly the paths a future refactor would silently break, and two of them are only reachable from the new profile context, so a dashboard-focused reviewer wouldn't notice.

**Fix:** Three small unit tests. The `onError` one is the highest value — assert `router.replace` is called with `{ pathname: "/team/profile/[memberId]", query: { memberId: "123" } }` given a starting query of `{ memberId, orderId, jobId }`.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

The PR description reports the full suite green (`turbo run test` 0 failures, plus typecheck, lint and build) on the author's machine. That is not independently verified here.

---

## Tests

- ✅ `JobsTab.test.tsx` — asserts the pushed URL for both an assigned row and an available-from-queue row, and that `JobSidebar` is never rendered even when both row sets are populated (the mock would render a `job-sidebar` testid if it were).
- ✅ `HistoryTab.test.tsx` (new) — asserts the pushed URL, the `userId`-missing no-op, and that `JobSidebar` is never rendered.
- ✅ `MemberOrderSidebar.test.tsx` (new) — covers all four behaviours of the hook: params forwarded to the provider, close clears both params, group navigation drops `jobId`, copy-URL includes both params.
- ✅ `OrderJobs/index.test.tsx` (new) — covers default-to-current-job, default-to-`initialJobId`, and fallback when `initialJobId` isn't part of the order. Asserts both `initialSlide` and `slideTo`, which is the right pair since `initialSlide` alone only applies at mount.
- ⚠️ No test where the mocked `jobs` are out of `jobSequence` order — the case Issue 1 describes.
- ❌ No test for `useOrderSidePanelData`'s `onError` param-stripping (Issue 6).
- ❌ No test for `OrderManagementSidebar`'s optional-`setBatchName` branch (Issue 6).
- ❌ No test for `useOrderMoreOptionsDropdownList` hiding "Show group" (Issue 6).
- ❌ No test locking in `formatWordQuantity`'s `en-US` pinning (Issue 5).
- ⏭️ Suite not executed this session — see Validation Checks.
- Manual coverage: the author reports verifying all five ✅ testing notes locally, plus the two ❌ ones which are unreachable by construction.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Requirements 1, 1a, 2 and 2a are all met; the two supporting bug fixes are correct and necessary |
| Regression risk | ⚠️ Medium — three shared admin-panel files changed with no test coverage; dashboard paths verified by reading, not by running |
| Tests | ⚠️ Strong on the new profile surface, absent on the shared-code edits the project requires tests for |
| Code quality | ⚠️ Good structure and comments; URL-building inconsistent with the PR's own `onError` fix, and one unrelated shared-util change |
| Validation suite | ⏭️ Skipped — user opted out |
| Mergeable state | ⚠️ GitHub reports `clean`, but validation was not run this session |

---

## Recommendation

**Approve with suggestions** — the feature is correct and well-structured; nothing here blocks merge on its own. Before merging:

1. **Re-run the validation suite** (`test` / `typecheck` / `lint` / `build`) against the PR branch. It was skipped in this review, and CLAUDE.md treats any failure as a hard blocker.
2. **Address Issue 1** — make `targetJobIndex`'s fallback resolve against `sortedJobs` (use `initialSlideIndex`), or at minimum document the mismatch and raise a follow-up. This is the one finding that can put the panel on the wrong card.
3. **Add the missing tests from Issue 6**, especially for `useOrderSidePanelData`'s `onError` — it's the change most likely to be undone by a future "simplification".
4. **Decide on Issue 5** — either split the `formatWordQuantity` locale pin into its own ticket, or keep it and add the regression test plus a release-note line, since it changes number rendering for non-US browsers in both portals.
5. **Optional (Issues 2, 3, 4)** — switch the profile URL building to the object form for consistency, gate `<MemberOrderSidebar />` on `router.query.orderId`, and either sync the active tab to the URL or soften the "shareable link" claim in the description.
