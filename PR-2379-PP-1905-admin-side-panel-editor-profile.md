# PR Review: feature/PP-1905: Open admin side panel from editor profile job links

**PR:** https://github.com/Proofed/B2BWebserver/pull/2379
**Jira:** https://proofed.atlassian.net/browse/PP-1905
**Status:** ✅ Review resolved — all findings addressed or consciously deferred; validation suite green
**Branch:** `feature/PP-1905-admin-side-panel-editor-profile` → `develop`
**Size:** 18 files, +723 / −209 at review time; review fixes add a further 15 files changed, +519 / −49 (commit `596c189eb`)

---

## Resolution Summary

| # | Issue | Severity | Resolution |
|---|---|---|---|
| 1 | `initialJobIndex` / `currentJobIndex` index-space mismatch | medium | ✅ **Fixed** — fallback now resolves against `sortedJobs` via `initialSlideIndex`; 2 regression tests added |
| 2 | Panel URLs built by string concatenation | low | ✅ **Fixed** — object-form `{ pathname, query }` in all three call sites; 3 test files updated |
| 3 | Ungated master-reference query on every profile load | low | ✅ **Fixed** — via the report's own preferred upstream option: `useLanguageLocalizationOptions(enabled)`; 3 tests added |
| 4 | Shared `?orderId=&jobId=` link lands on Settings tab | low | ⏭️ **Deferred by design** — PR description now states the limitation instead of claiming shareability. Rationale below |
| 5 | `formatWordQuantity` locale pinning is unrelated scope | low | ✅ **Kept + locked** — existing test renamed to state the intent; drive-by declared in the PR description |
| 6 | Three shared-code behaviour changes ship without tests | low | ✅ **Fixed** — 7 new tests across 3 suites (2 new files, 1 extended) |

**Net: 5 of 6 fixed, 1 deferred with a documented reason.**

---

## Corrections to the original review

Three points in the review as first written were inaccurate. Recording them so the report is not cited later as-is:

1. **Issue 1 — the `-1` edge does not exist.** The review implied `currentJobIndex` could land on `-1`. Both `initialSlideIndex` and `initialJobIndex` already clamp (`idx >= 0 ? idx : 0` / `: null`), so the real defect was purely the *index space* (raw query array vs `sortedJobs`), not an unguarded `-1`. The fix and its rationale stand; the framing was wrong.
2. **Issue 5 — a locking test already existed.** `formatWordQuantity.test.ts` already asserted `formatWordQuantity(1000000, true) === "1,000,000 words"`. The review's suggested test was therefore a duplicate. What was actually missing was any statement of *intent* — the assertion read as an ordinary large-number case, so a future developer could "fix" it for their locale. It was renamed to `"uses en-US digit grouping regardless of host locale"` with a comment naming the en-IN failure mode. Additionally, the review's claim that the change "ships to the customer portal as well" is technically true of the package but has **zero practical effect**: `formatWordQuantity` has no customer-portal call sites.
3. **Issue 6 — the batch-name render branch is not in this PR.** The review asked for a test that "the batch name renders as plain text (not a `RawButton`) when the handler is absent". That branch lives in `DetailedOrderInfo.tsx`, which this PR does not touch. The change that *is* in this PR is the prop becoming optional and the handler being conditionally forwarded, so the test asserts the forwarding contract instead.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Clicking any job in the assigned jobs list opens the admin side panel | `JobsTab.onRowClick` looks the clicked job up in `flatAssignedRows`/`flatAvailableRows` and pushes `?orderId=<job.orderId>&jobId=<job.id>`; `MemberOrderSidebar` (new page partial) renders `OrderSidebarProvider` + `OrderManagementSidebar` — the same panel as the order management dashboard | ✅ Addressed |
| 1. Clicking any job in job history opens the admin side panel | `HistoryTab.onRowClick` pushes `?orderId=<item.original.order_id>&jobId=<item.original.id>`; `order_id` is a required field on `JobHistoryRowItem` and is populated in `user-job-history/utils.ts` (`order_id: job.orderId.toString()`) | ✅ Addressed |
| 1a. The user/client side panel must not open from this context | `JobSidebar` import and render removed from both `JobsTab` and `HistoryTab`; a repo-wide grep under `components/pages/team-members` shows no remaining reference outside test mocks. `CalendarTab` and `SettingsTab` have no job links | ✅ Addressed |
| 2. Panel must default to the job card for the job that was clicked | New optional `initialJobId` on `OrderSidebarProviderProps` → context → `OrderJobs` resolves it to `initialJobIndex` and uses it for both `initialSlide` and the `slideTo` effect | ✅ Addressed |
| 2a. Order management dashboard must keep defaulting to the current job | `components/pages/admin-area/orders/index.tsx` does not pass `initialJobId`, so `targetJobIndex` falls back to `initialSlideIndex` — the order's current job, now resolved against the same sorted array the slides render from | ✅ Addressed (hardened by the Issue 1 fix) |
| Testing note 6 — clicking a profile job while the client panel is open must open the admin panel | Unreachable by construction: the client panel can no longer be rendered on the profile at all | ✅ Addressed |
| Testing note 7 — client panel must not be openable from any job link on the profile | Covered by the removals above; asserted by `JobsTab.test.tsx` and `HistoryTab.test.tsx` ("never renders the user/client JobSidebar") | ✅ Addressed |

**Changes beyond the Jira scope:**

- `services/jobs/index.ts` and `services/customers/index.tsx` — relative `"api/jobs"` / `"api/customers"` URLs made absolute. **In practice required** for the feature: axios resolves a relative URL against the current directory, so `api/jobs` worked from `/orders` (→ `/api/jobs`) but 404s from `/team/profile/123` (→ `/team/profile/api/jobs`). Confirmed `apiRoutes.jobs === "/api/jobs"` and that no other relative axios URL remains anywhere in `creative-portal` or `packages/shared`.
- `useOrderSidePanelData` `onError` — `router.replace("")` → explicit `{ pathname, query }`. Also effectively required; see Architecture Analysis. **Now covered by tests** (Issue 6).
- `packages/shared/utils/formatWordQuantity.ts` — locale pinned to `en-US`. Genuine drive-by; kept deliberately and declared in the PR description (Issue 5).
- `OrderManagementSidebar.setBatchName` widened to optional, and `useOrderMoreOptionsDropdownList` now hides "Show group" when no handler is supplied. Both are consequences of reusing the panel outside the dashboard. **Now covered by tests** (Issue 6).
- `useLanguageLocalizationOptions` gained an `enabled` parameter (Issue 3 fix). Backwards-compatible: defaults to `true`.

---

## Architecture Analysis

The approach is the right one and matches Orlin's guidance on the ticket ("we would need to create a new component"): rather than teaching `JobSidebar` to behave like the admin panel, the PR reuses the *existing* admin panel (`OrderSidebarProvider` + `OrderManagementSidebar`) behind a thin new page partial, and adds one optional input (`initialJobId`) to select the opening card. Panel state lives in the URL, matching the dashboard's `?orderId=` convention.

Things verified rather than assumed:

- **The `router.replace("")` fix is real.** In Next 14, `resolveHref` builds its base from `router.pathname`, not `router.asPath` (`node_modules/next/dist/client/resolve-href.js:40`). So on `/orders` (pathname `/`) `""` resolves to `/` and correctly clears the query — which is why the dashboard's existing close works. On `/team/profile/[memberId]` it resolves to the literal un-interpolated pattern, and `router.change` then throws "missing query values … to be interpolated properly". The replacement preserves `memberId` and every other route/query param, and still produces `/` on the dashboard, so there is no regression there.
- **No privilege escalation.** The profile page is gated on `[Admin, Superadmin, ServiceDelivery, ServiceSupport]`; the orders dashboard on those plus `Returner`. The profile's role set is a strict subset, so no role gains access to order-management actions it couldn't already reach via `/orders`.
- **Click-outside doesn't fight the row click.** `useOrderManagementSidebar`'s document listener skips targets with a `TR` ancestor (`hasAllowedParentTag(..., ["TR"], ["THEAD"])`), and both `JobTable` and `HistoryTable` render real `<table>/<tr>` markup via react-table. So clicking a second job row while the panel is open switches orders instead of closing the panel — same as the dashboard.
- **The always-mounted provider doesn't fetch order data when closed.** `useOrderByIdQuery`, `useOrderSupportDocumentsQuery`, `useOrderJobTaskQuery` and `useCustomerQuery` are all gated on `deferredOrderId`, and `useJobsQuery` on `!!orderId`. The one ungated query (Issue 3) is now gated too.
- **Empty "more options" dropdown is not a risk.** `OrderSidebarHeader` guards with `!!moreDropdownItems?.length`, so a cancelled ungrouped order on the profile renders no dropdown rather than an empty one. This is now asserted by a test (Issue 6).

`MemberOrderSidebar` follows the repo's `index.tsx`-is-UI-only convention (state and callbacks in a sibling `hooks.ts`). It has no `types.ts`, which is fine — it takes no props, matching `CalendarTab`.

---

## Issues Found

### 1. `initialJobIndex` and `currentJobIndex` are indices into two differently-ordered arrays ✅ FIXED

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/index.tsx]**

**Function/Class:** OrderJobs

**Severity:** medium

**Problem:** `targetJobIndex = initialJobIndex ?? currentJobIndex` combined two indices not measured against the same array. `initialJobIndex` is a `findIndex` over `sortedJobs` — `jobs` sorted by `order.jobSequence`. `currentJobIndex` came from the provider and is a `findIndex` over the raw react-query `jobs` array, memoised on `[order, jobs]`. `OrderJobs` sorts in place (`jobs.sort(...)` mutates), so the array identity never changes and the provider's memo never recomputes against the sorted order. The slides, meanwhile, are rendered from `sortedJobs`. The whole reason `sortJobsByJobSequence` exists is that the API does not guarantee `jobSequence` order, so the two can genuinely disagree.

**Impact:** On the fallback path — every order-management-dashboard open, plus every profile open where `jobId` isn't in the order (group navigation, a stale link) — the panel could land on the wrong job card whenever the jobs endpoint returned a different order than `jobSequence`.

**Resolution:** ✅ **Fixed as recommended.** The fallback is now the file's own `initialSlideIndex`, which resolves `order.currentJobId` against `sortedJobs`, and the redundant `?? initialSlideIndex` tail on `initialSlide` was dropped:

```tsx
// Card the panel opens on: the explicitly requested job (e.g. a
// job link on the editor profile) wins over the order's current
// job. Both indices resolve against sortedJobs — the array the
// slides below are rendered from. The context's currentJobIndex
// must not be used here: it indexes the raw query array, which
// this component sorts in place, so the two disagree whenever the
// API returns jobs out of jobSequence order.
const targetJobIndex = initialJobIndex ?? initialSlideIndex;
```

Two regression tests were added to `OrderJobs/index.test.tsx` (5 tests total, all passing):

- jobs returned **out of `jobSequence` order** with the context supplying `currentJobIndex: 0` — asserts `initialSlide: 2`, i.e. the sorted-array position wins. This test fails against the old expression.
- `order.currentJobId` **absent from the jobs array** — asserts `initialSlide: 0` rather than a negative index.

**Note:** the review's `-1` framing was wrong — see Corrections above. Both index helpers already clamped; the defect was the index space alone.

### 2. Panel URLs are built by string concatenation and drop every other query param ✅ FIXED

**[File: apps/creative-portal/components/pages/team-members/profile/[memberId]/partials/MemberOrderSidebar/hooks.ts]**

**Function/Class:** useMemberOrderSidebar — `setActiveOrderId` (plus both `onRowClick` handlers)

**Severity:** low

**Problem:** `setActiveOrderId` and both `onRowClick` handlers in `JobsTab`/`HistoryTab` rebuilt the URL as `` `${appRoutes.teamMembersProfile(userId)}?orderId=…` ``, discarding anything else on the query string. The sibling fix in this same PR — `useOrderSidePanelData`'s `onError` — deliberately does the opposite, spreading `router.query` and deleting only `orderId`/`jobId`.

**Impact:** No user-visible breakage today; the profile route carries no other query params. It becomes a real bug the moment one is added (a tab key, a filter, a `?new` flag), and the inconsistency inside a single PR made the safer pattern easy to miss later.

**Resolution:** ✅ **Fixed as recommended.** All three call sites use the object form, spreading `router.query` and deleting only what they own:

- `MemberOrderSidebar/hooks.ts` — `jobId` always dropped, `orderId` dropped only on close.
- `JobsTab/index.tsx` and `HistoryTab/index.tsx` — `router.push({ pathname: router.pathname, query: { ...router.query, orderId, jobId } }, undefined, { scroll: false })`.

The now-unused `appRoutes` imports were removed from both tabs. The three corresponding test files were updated to the object form (`pathname: "/team/profile/[memberId]"`, `query: { memberId: "123" }`), which also makes them assert param preservation rather than a concatenated string.

### 3. The order panel now mounts on every profile page load, firing one ungated query ✅ FIXED

**[File: packages/shared/hooks/useLanguageLocalizationOptions.ts]**

**Function/Class:** useLanguageLocalizationOptions / OrderEditDetailsModal

**Severity:** low

**Problem:** `<MemberOrderSidebar />` renders unconditionally, so `OrderSidebarProvider` → `OrderManagementSidebar` → `OrderManagementModals` → `OrderEditDetailsModal` mounts on every profile visit even with no `orderId`. `OrderEditDetailsModal` calls `useLanguageLocalizationOptions()`, which issued `useMasterReferenceQuery(ReferenceName.LanguageLocalization)` with no `enabled` guard. Everything genuinely order-scoped is correctly gated, so this was the one that leaked.

**Impact:** One extra master-reference request per profile page load (react-query caches it for the session), plus the mount cost of the modal tree on a page that mostly won't open an order.

**Resolution:** ✅ **Fixed via the report's own preferred option — the upstream guard, not the conditional mount.** The report flagged that gating `<MemberOrderSidebar />` on `router.query.orderId` would unmount `OrderSidebar` the instant `orderId` clears and skip the panel's close transition, and noted the cleaner fix "belongs upstream … which would also benefit the dashboard". That is what was done:

```ts
// `enabled` lets always-mounted callers (e.g. a modal that renders
// nothing until opened) defer the request until the options are
// actually needed. Defaults to true so existing callers are
// unaffected; the languageSelectItems fallback covers the gap while
// the deferred request is in flight.
const useLanguageLocalizationOptions = (enabled = true) => { … }
```

`OrderEditDetailsModal` now calls `useLanguageLocalizationOptions(isOpen)`. The default keeps every other caller untouched, and the dashboard gets the same saving. Three hook tests were added to `useLanguageLocalizationOptions.test.ts` (12 → 15, all passing): fetches by default, defers when disabled, and still returns the `languageSelectItems` fallback while disabled.

### 4. A shared `?orderId=&jobId=` link lands on the Settings tab ⏭️ DEFERRED (deliberate)

**[File: apps/creative-portal/components/pages/team-members/profile/[memberId]/index.tsx]**

**Function/Class:** TeamProfilePage / Tabs

**Severity:** low

**Problem:** `useTabs` keeps the active tab in local React state with no URL sync, and `items[0]` is Settings. Within a session the tab survives (a same-route `router.push` re-renders rather than remounts), so clicking a job never bounces the admin out of Jobs/History — good. But on a refresh, or on a link opened by another admin, the panel opens on top of the Settings tab.

**Impact:** Cosmetic. The panel content is correct; only the page behind it is the wrong tab. Nothing in PP-1905 asks for shareable links — that framing came from a PR-description claim, not from the ticket.

**Resolution:** ⏭️ **Not fixed — deliberately deferred, with the reason recorded in the PR description.** Rationale:

- **It is not a PP-1905 requirement.** The ticket asks for the admin panel to open on the clicked job. It does. Deep-link tab restoration is a separate behaviour of the profile page that predates this PR.
- **The fix is not local.** Syncing the tab key into the URL means converting `useTabs` to controlled mode on this page, choosing a canonical `?tab=` vocabulary, and back/forward-proofing it. That touches a shared hook and every tab on the profile — a materially larger change than the feature under review, landing in a PR that is already 18 files.
- **The honest alternative was taken instead.** Rather than half-fix it, the PR description no longer claims shareable/refresh-survivable links and now states the limitation explicitly, so QA does not file it as a defect and a follow-up can pick it up with proper scope.

Nothing regresses: in-session behaviour — the only path PP-1905 exercises — is correct.

### 5. `formatWordQuantity` locale pinning is unrelated scope and changes rendering for all users ✅ KEPT + LOCKED

**[File: packages/shared/utils/formatWordQuantity.ts]**

**Function/Class:** formatWordQuantity

**Severity:** low

**Problem:** The change is motivated in the PR description as a test fix ("its unit test failed on en-IN systems"), but `new Intl.NumberFormat()` reads the *browser's* locale at runtime, not just the test host's. Pinning to `en-US` changes what real users see. It has nothing to do with PP-1905 and sits in `packages/shared`.

**Impact:** A user on an `en-IN` browser previously saw `10,00,000 words` and now sees `1,000,000 words`. That is arguably the *correct* outcome — it matches `formatPriceWithCurrencySymbol` and `formatCurrency`, both of which already pin `en-US` — but it is a product decision.

**Resolution:** ✅ **Kept deliberately, and the intent is now locked by a test.** Two corrections to the finding first (see Corrections above): a locking assertion already existed, and `formatWordQuantity` has **zero customer-portal call sites**, so the "ships to the customer portal as well" concern is theoretical.

What was actually missing was a statement of *intent* — the existing assertion read as an ordinary large-number case, so a future developer hitting it on an en-IN machine could "fix" it by unpinning the locale and reintroduce the bug. The test was renamed and commented:

```ts
// Guards the pinned en-US locale: an unpinned Intl.NumberFormat
// groups this as "10,00,000" on an en-IN host, so this assertion
// is what keeps grouping independent of who is viewing.
it("uses en-US digit grouping regardless of host locale", () => {
  expect(formatWordQuantity(1000000, true)).toBe("1,000,000 words");
});
```

Splitting it into its own ticket was considered and rejected: the one-line change is what makes the suite deterministic on non-US machines, so extracting it would leave this PR red on any en-IN developer's box. Instead it is declared as a drive-by in the PR description so it is visible in review and release notes. 9 tests in the file, all passing.

### 6. Three behaviour changes outside the profile ship without tests ✅ FIXED

**[File: apps/creative-portal/components/pages/admin-area/orders/hooks.tsx + OrderManagementSidebar + useOrderMoreOptionsDropdownList]**

**Severity:** low

**Problem:** The 23 new tests covered the profile-side additions well, but the three edits to shared admin-panel code had none — including `useOrderSidePanelData`'s `onError` param-stripping, the change that actually prevents a thrown error and the one most likely to regress if someone "simplifies" it back to `router.replace("")`.

**Impact:** CLAUDE.md requires tests for new code. These are exactly the paths a future refactor would silently break, and two of them are only reachable from the new profile context.

**Resolution:** ✅ **Fixed — 7 tests across 3 suites.**

**`apps/creative-portal/components/pages/admin-area/orders/hooks.test.tsx` (new, 3 tests)** — the highest-value one. Mocks the nine service modules the hook pulls in, captures the options object handed to `useJobsQuery`, and invokes `onError` directly:

- closes the panel by stripping **only** `orderId`/`jobId`, asserting `router.replace({ pathname: "/team/profile/[memberId]", query: { memberId: "123" } }, undefined, { scroll: false })` — exactly the assertion the review asked for, and the one that fails if anyone reverts to `router.replace("")`.
- preserves unrelated params (`tab=history` survives the close).
- surfaces the "This order was not found." error toast.

**`.../OrderManagementSidebar/index.test.tsx` (new, 2 tests)** — stubs `next/dynamic` (every child of the component is a dynamic import) to capture the props each child receives, then locates the one call carrying an `onBatchNameClick` key:

- with `setBatchName` supplied (dashboard): the handler is forwarded, and invoking it closes the panel first (`setActiveOrderId(undefined)`) before applying the batch filter.
- without it (editor profile): no handler is forwarded at all.

Note the review's requested assertion — plain text vs `RawButton` — belongs to `DetailedOrderInfo.tsx`, untouched by this PR; the forwarding contract is what changed and is what is asserted.

**`.../OrderManagment/useOrderMoreOptionsDropdownList.test.tsx` (extended, 6 → 8 tests)** — the module-level handler mock became a `let` reset in `beforeEach`, so both branches are exercised:

- a grouped **Live** order with no `onOrderGroupClick` hides "Show group" while the rest of the menu is unaffected ("Edit Order Details" still present).
- a grouped **Cancelled** order with no handler yields an **empty** menu — the case that proves `OrderSidebarHeader`'s `!!moreDropdownItems?.length` guard is what stops an empty dropdown rendering on the profile.

---

## Validation Checks

Run on the PR branch at commit `596c189eb`, with `LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8` — the development host is en-IN, and `formatWordQuantity` is not the only suite that reads the host locale.

**`npx turbo run test typecheck lint build` → 18 successful, 18 total.**

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 3,690 passing, 4 skipped, 0 failures — `@proofed/shared` 1,322 / 133 files · `creative-portal` 1,767 / 194 files · `customer-portal` 336 / 37 files · `wysiwyg-editor` 265 / 23 files |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces |
| `npx turbo run lint` | ✅ | 0 errors, 0 warnings |
| `npx turbo run build` | ✅ | All four workspaces build clean |

Two earlier passes each failed on `prettier/prettier` errors — 5 in the new `useLanguageLocalizationOptions.test.ts` block, then 2 more on union-type line wrapping in `useOrderMoreOptionsDropdownList.test.tsx` and `OrderManagementSidebar/index.test.tsx`. Both rounds were in newly written test code, both fixed, and the suite re-run to green each time. Recorded because it is the review's own point in miniature: new test files are not exempt from the gates, and eyeballing does not catch this class.

**Pre-existing, not fixed (deliberate):** `admin-area/orders/hooks.tsx` carries an unused-import lint warning on `isBefore`. The file is byte-identical to `origin/develop` at that hunk (last touched by `c71cdc8f4`), so it is not this PR's to fix — per the team's rule that a feature PR does not absorb unrelated pre-existing failures.

---

## Tests

- ✅ `JobsTab.test.tsx` — asserts the pushed URL for both an assigned row and an available-from-queue row, and that `JobSidebar` is never rendered even when both row sets are populated. **Updated** to the object URL form (Issue 2).
- ✅ `HistoryTab.test.tsx` — asserts the pushed URL, the `userId`-missing no-op, and that `JobSidebar` is never rendered. **Updated** to the object URL form (Issue 2).
- ✅ `MemberOrderSidebar.test.tsx` — covers all four behaviours of the hook: params forwarded to the provider, close clears both params, group navigation drops `jobId`, copy-URL includes both params. **Updated** to the object URL form (Issue 2).
- ✅ `OrderJobs/index.test.tsx` — default-to-current-job, default-to-`initialJobId`, fallback when `initialJobId` isn't part of the order, **plus 2 new**: out-of-`jobSequence`-order API response, and a missing current job. 5 passing (Issue 1).
- ✅ `admin-area/orders/hooks.test.tsx` **(new)** — `onError` param-stripping, unrelated-param preservation, error toast. 3 passing (Issue 6).
- ✅ `OrderManagementSidebar/index.test.tsx` **(new)** — optional `setBatchName` forwarded / not forwarded. 2 passing (Issue 6).
- ✅ `useOrderMoreOptionsDropdownList.test.tsx` **(extended)** — "Show group" hidden with no handler, and the empty-menu Cancelled case. 8 passing (Issue 6).
- ✅ `useLanguageLocalizationOptions.test.ts` **(extended)** — `enabled` gating in both directions plus the fallback while disabled. 15 passing (Issue 3).
- ✅ `formatWordQuantity.test.ts` — intent-stating rename on the en-US grouping assertion. 9 passing (Issue 5).
- Manual coverage: the author reports verifying all five ✅ testing notes locally, plus the two unreachable-by-construction ones.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Requirements 1, 1a, 2 and 2a met; the two supporting bug fixes are correct and necessary; the Issue 1 index-space bug is fixed and locked |
| Regression risk | ✅ Low — all three shared admin-panel changes now have targeted tests; dashboard fallback path hardened rather than merely preserved |
| Tests | ✅ Strong on the new profile surface, and now on every shared-code edit |
| Code quality | ✅ URL building consistent across the PR; the one drive-by is declared and locked by a test |
| Validation suite | ✅ test / typecheck / lint / build all green (18/18) |
| Open items | ⏭️ Issue 4 only (tab not restored on a deep link) — deferred by design, documented in the PR description |

---

## Recommendation

**Approve.** All five actionable findings are fixed, the sixth is consciously deferred with its reason recorded in the PR description, and the full validation suite is green.

Post-merge follow-up (not blocking):

- **Issue 4** — if deep-linking to the profile becomes a real workflow, sync the active tab into the URL (`?tab=`) via `useTabs`' controlled mode, as its own ticket.
- **`formatWordQuantity`** — worth a release-note line, since it changes digit grouping for non-US browsers.
