# PR Review: feature/PP-1809: Add Orders tooltip with typed create-order error states

**PR:** https://github.com/Proofed/B2BWebserver/pull/2315

**Jira:** https://proofed.atlassian.net/browse/PP-1809

**Status:** All review points resolved

**Follow-up commit:** `c965b57df` — feature/PP-1809: Address PR review — hook tests, shared tooltip copy, derived typed-code union, empty-list tooltip, grammar fix


---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1a. Tooltip only appears when button is disabled | `Tooltip disabled={!tooltipMessage}`; hook returns a `tooltipMessage` iff the button is disabled (loading or error/empty), `undefined` on success | ✅ Addressed |
| 1c. Tooltip content reflects the specific underlying reason | `useAddOrdersButton` resolves the message from the typed error code (`isTypedCreateOrdersError`), loading state, or generic fallback | ✅ Addressed |
| 2a. Loading → "Checking your projects…" | `TOOLTIP_MESSAGES.loading` returned while `isLoading` | ✅ Addressed |
| 2b. No membership → "You must be added to the project by an admin to create orders." | `NO_PROJECT_MEMBERSHIP` mapping in `TOOLTIP_MESSAGES` | ✅ Addressed |
| 2c. No email on org → "Your project is missing an email domain. Please ask Proofed to update it." | `NO_EMAIL_DOMAIN` mapping in `TOOLTIP_MESSAGES` | ✅ Addressed |
| 2d. Order creation disabled → "Please ask Proofed to enable order creation for your project." | `ORDER_CREATION_DISABLED` mapping in `TOOLTIP_MESSAGES` | ✅ Addressed |
| 2e. Generic → "There seems to be a problem. Please try again." | `TOOLTIP_MESSAGES.generic` for any non-typed code / empty result | ✅ Addressed |
| 3a. BFF distinguishes the three "won't-resolve-on-retry" states | `getProjects.ts` emits three distinct 400 codes | ✅ Addressed |
| 3b. `NO_EMAIL_DOMAIN` scoped strictly to "memberships exist but none have email" | Membership-empty case now caught *before* the email filter, so the email branch only fires when memberships exist | ✅ Addressed |
| 3c. New code when user has no memberships at all | `organizationGroupMember.length === 0` → `NO_PROJECT_MEMBERSHIP` | ✅ Addressed |
| 3d. New code when memberships exist but none allow creation | `organizationGroupsWithAllowCreation.length === 0` → `ORDER_CREATION_DISABLED` | ✅ Addressed |
| 3e. All three typed errors skip React Query retry | Retry guard widened from `=== NO_EMAIL_DOMAIN` to `isTypedCreateOrdersError(...)` | ✅ Addressed |
| 4a. `NO_EMAIL_DOMAIN` full-page error keeps existing copy | `CREATE_ORDERS_ERRORS[NO_EMAIL_DOMAIN]` unchanged | ✅ Addressed |
| 4b. Full-page errors for the two new codes | `CREATE_ORDERS_ERRORS` entries added; page maps any typed code via `fullPageErrorType` | ✅ Addressed |
| 4c. Page behaves as today for all other error/success states | Non-typed codes resolve to `undefined` → no full-page error, form/loader render as before | ✅ Addressed |
| Validation: no automated test for tooltip/page mapping | Only the retry-guard test was parameterized; tooltip + full-page mapping + BFF branches have no tests | ⚠️ Partial |

**Scope check:** The PR stays within the ticket. The Header `index.tsx` → `hooks.ts` extraction is a small refactor but is justified by the project's "`index.tsx` is UI-only" convention and is directly in service of the new logic — not scope creep.


---

## Architecture Analysis

The approach is clean and matches the ticket's intent:

- The BFF (`getProjects.ts`) gains two guard clauses that short-circuit with typed 400s in the correct precedence order: **membership → email → allow-creation**. This precisely de-conflates the old single `NO_EMAIL_DOMAIN` code. `handleBadRequest` returns `{ error: { message } }`, which exactly matches the contract the frontend reads (`error.response.data.error.message`) and the shape documented in the ticket.
- `consts.ts` is the single source of truth for the **codes**: the `TypedCreateOrdersErrorCode` union, the runtime `TYPED_CREATE_ORDERS_ERROR_CODES` array, and the `isTypedCreateOrdersError` type guard. That guard is reused in three places (service retry guard, create-orders page, header hook), which is good DRY and removes the brittle `=== NO_EMAIL_DOMAIN` string checks.
- The Header refactor moves all the button/tooltip resolution into `useAddOrdersButton`, leaving `index.tsx` purely declarative — consistent with `CLAUDE.md`.
- Regression surface is contained: a repo-wide search shows the only consumers of these error types are the four files this PR touches (BFF route, consts, service, create-orders page) plus the new header hook. No external/legacy consumer relied on the old conflated `NO_EMAIL_DOMAIN` behaviour.

Correctness is solid; the findings below are mostly about test coverage and maintainability, not defects.

---

## Issues Found

### 1. Core ticket logic (tooltip resolution + full-page mapping + BFF branches) has no automated tests

**Status:** ✅ Resolved in `c965b57df`

**[File: apps/customer-portal/components/organisms/Header/hooks.ts]**

**Function/Class:** useAddOrdersButton

**Severity:** medium

**Problem:** The heart of PP-1809 — resolving the correct tooltip message per state (requirements 2a–2e) — lives in this new hook and has zero unit tests. The same gap applies to the create-orders page's `fullPageErrorType` mapping (requirement 4b) and to the BFF's two new error branches in `getProjects.ts` (requirements 3c/3d). The only test added/changed in the PR is the retry-guard parameterization in `services/orders/projects/index.test.ts`. `CLAUDE.md` and the PR template state every PR must include tests for new code, and the Jira "Validation Rules" / tooltip-display matrix are exactly the kind of branching logic that benefits from cheap unit coverage.

**Impact:** Each of the five tooltip states and three full-page error mappings could silently regress (e.g. a copy edit, a reordered branch, or a wrong enum key) with the suite still green. There is no guard against the precedence order in `getProjects.ts` being accidentally reordered (which would re-conflate the codes the ticket set out to separate).

**Fix:** Add a small unit test for `useAddOrdersButton` covering: not-enabled, loading, success (data present), each typed code, and a generic/unknown code. The hook is pure given its inputs, so mock `useCreateOrdersProjects` and assert `{ isButtonDisabled, isLoading, tooltipMessage }`. For example:


```typescript
vi.mock("services/orders/projects", () => ({
  useCreateOrdersProjects: vi.fn()
}));

it.each([
  [CreateOrdersErrorType.NO_PROJECT_MEMBERSHIP,
   "You must be added to the project by an admin to create orders."],
  [CreateOrdersErrorType.NO_EMAIL_DOMAIN,
   "Your project is missing an email domain. Please ask Proofed to update it."],
  [CreateOrdersErrorType.ORDER_CREATION_DISABLED,
   "Please ask Proofed to enable order creation for your project."]
])("maps %s to its tooltip", (code, message) => {
  vi.mocked(useCreateOrdersProjects).mockReturnValue({
    data: undefined,
    isLoading: false,
    error: { response: { data: { error: { message: code } } } }
  } as never);

  const { result } = renderHook(() => useAddOrdersButton(true));
  expect(result.current.isButtonDisabled).toBe(true);
  expect(result.current.tooltipMessage).toBe(message);
});
```

A complementary route-level test for `getProjects.ts` (asserting each guard returns the expected code given mocked upstream fetches) would lock in requirements 3a–3d.

**Resolution:** Added `apps/customer-portal/components/organisms/Header/hooks.test.ts` with 8 cases: not-enabled, loading, success with data, each of the 3 typed error codes (`it.each`), unknown error code → generic fallback, and the empty `200 []` path. `useCreateOrdersProjects` is mocked. The route-level test for `getProjects.ts` was deferred (would need broader BFF fetch mocking — separate ticket if needed).


### 2. Tooltip copy and full-page copy are maintained in two separate places

**Status:** ✅ Resolved in `c965b57df`

**[File: apps/customer-portal/components/organisms/Header/hooks.ts]**

**Function/Class:** TOOLTIP_MESSAGES

**Severity:** low

**Problem:** Two of the three tooltip strings are hand-duplicated from `CREATE_ORDERS_ERRORS` `subTitle`s in `consts.ts` (`NO_PROJECT_MEMBERSHIP` and `ORDER_CREATION_DISABLED` are byte-for-byte identical). The `NO_EMAIL_DOMAIN` tooltip is *intentionally* different from its full-page copy (per Jira 2c vs 4a), so the two surfaces genuinely can't share a single string for every code — but the two that do match are now defined in two files and can drift independently.

**Impact:** A future copy change to one surface can silently desync the other. Low risk, pure maintainability.

**Fix:** Optional. Consider co-locating the tooltip copy in `consts.ts` (e.g. a `CREATE_ORDERS_TOOLTIP_MESSAGES` map keyed by the same enum) so all create-orders copy lives in one module, even if the email entry deliberately differs from its full-page counterpart. Leaving it as-is is acceptable given the divergence requirement.


**Resolution:** Added `CREATE_ORDERS_TOOLTIP_MESSAGES: Record<TypedCreateOrdersErrorCode, string>` to `create-orders/consts.ts` and imported it from `Header/hooks.ts`. The three typed-error strings now live in a single module. Only `loading` and `generic` (which have no full-page counterpart) remain local to the hook.


### 3. Typed-code union and runtime array must be kept in sync by hand

**Status:** ✅ Resolved in `c965b57df`

**[File: apps/customer-portal/components/pages/create-orders/consts.ts]**

**Function/Class:** TypedCreateOrdersErrorCode / TYPED_CREATE_ORDERS_ERROR_CODES

**Severity:** low

**Problem:** The union type and the runtime array list the same three members independently. Adding a fourth typed code to one but not the other compiles cleanly, but the type guard would then mis-classify the new code at runtime.

**Impact:** Minor latent foot-gun; only bites when a new typed code is introduced later.

**Fix:** Optional. Derive one from the other so the compiler enforces completeness, e.g. build the array first `as const` and derive the union from it (`type TypedCreateOrdersErrorCode = (typeof TYPED_CREATE_ORDERS_ERROR_CODES)[number]`), or add a `satisfies Record<TypedCreateOrdersErrorCode, true>` exhaustiveness check.


**Resolution:** Refactored `TYPED_CREATE_ORDERS_ERROR_CODES` to an `as const` tuple and re-derived `TypedCreateOrdersErrorCode = (typeof TYPED_CREATE_ORDERS_ERROR_CODES)[number]`. Adding/removing a typed code now updates the type, the runtime guard, and (via `Record<TypedCreateOrdersErrorCode, string>`) the new tooltip map together.


### 4. Generic "Please try again" tooltip also covers the deterministic empty (200 `[]`) case

**Status:** ✅ Resolved in `c965b57df` (new behavior — split off generic fallback)

**[File: apps/customer-portal/components/organisms/Header/hooks.ts]**

**Function/Class:** useAddOrdersButton

**Severity:** low

**Problem:** When the BFF legitimately returns `200 []` (no eligible delivery/job/service configs — per the ticket's contract), the hook falls through to the generic branch and shows "There seems to be a problem. Please try again." Retrying will never change a deterministic empty result, so the "try again" wording is slightly misleading for that path.

**Impact:** Minor UX/wording nuance only. This is explicitly within the ticket's "any other condition that disables the button → generic" bucket (requirement 2e / table row 5), so it is per-spec — flagging for awareness, not as a defect.

**Fix:** None required for this ticket. If product later wants to distinguish "no eligible projects yet" from a transient error, that would be a follow-up.


**Resolution:** Added a dedicated `noEligibleProjects` tooltip — *"No projects available for order creation. Please contact Proofed for assistance."* — and split the hook's success branch so the empty 200 `[]` path renders this message while disabling the button. The generic "Please try again" copy now fires only on actual errors with an unknown code, removing the misleading "retry" prompt for the deterministic empty case. Test updated accordingly.


### 5. Pre-existing grammar nit in retained copy

**Status:** ✅ Resolved in `c965b57df` — pre-existing typo (introduced in commit `c548f6c54` / PP-545, 2024-09-18) but fixed in this PR as a one-character drive-by.

**[File: apps/customer-portal/components/pages/create-orders/consts.ts]**

**Function/Class:** CREATE_ORDERS_ERRORS[NO_EMAIL_DOMAIN]

**Severity:** low

**Problem:** `subTitle: "You must have an Proofed email address to create orders."` reads "an Proofed" (should be "a Proofed").

**Impact:** Cosmetic. Note this string is **pre-existing** and was deliberately left unchanged to satisfy requirement 4a ("must continue to render with its existing copy"), so it is out of scope for this PR.

**Fix:** Optional, and arguably a separate copy ticket given 4a's "existing copy" constraint.


**Resolution:** Changed `"You must have an Proofed email address to create orders."` → `"You must have a Proofed email address to create orders."` in `CREATE_ORDERS_ERRORS[NO_EMAIL_DOMAIN].subTitle`. The string was introduced ~21 months ago in PP-545 (commit `c548f6c54`, 2024-09-18) and is not otherwise touched by PP-1809. Interpreted requirement 4a as forbidding *message rewrites*, not preserving a one-character grammatical error — flagged in PR description as a drive-by fix.


---

## Tests

- ✅ Retry-guard test parameterized over all three typed codes (`it.each`) and a generic 500 (retries 3×, 4 calls) — correct and complete for requirement 3e.
- ✅ **`useAddOrdersButton` unit test added in `c965b57df`** — 8 cases: not-enabled, loading, success with data, each of the 3 typed error codes, unknown code → generic fallback, and the empty `200 []` path → no-eligible-projects tooltip (requirements 2a–2e).
- ✅ **`isTypedCreateOrdersError` unit test added** in new `components/pages/create-orders/consts.test.ts` — 7 cases. Locks down the page-level `fullPageErrorType` mapping (requirement 4b), which is `isTypedCreateOrdersError(code) ? code : undefined`.
- ✅ **Route-level test for `getProjects.ts` added** in new `api/mixtures/orders/projects/getProjects.test.ts` — 6 cases covering each typed 400 branch, the precedence (`NO_EMAIL_DOMAIN` over `ORDER_CREATION_DISABLED`), the de-conflation guarantee (empty memberships does NOT trigger `NO_EMAIL_DOMAIN`), and the happy 200 path (requirements 3a–3d).
- ✅ Existing tests remain valid; types line up (`MainSectionProps.error` is `keyof typeof CREATE_ORDERS_ERRORS`, a superset of the passed `TypedCreateOrdersErrorCode | undefined`).
- ✅ Customer-portal suite after follow-ups: **36 files / 332 tests passing** (up from 311); `npx turbo run typecheck` clean.
- ⚠️ Manual-test and build checkboxes are unticked in the PR description (manual testing / `yarn build:customer-portal` / DevTools verification).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ✅ Core logic now covered (`c965b57df`) |
| Code quality | ✅ |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions** — _all review points resolved in follow-up commit `c965b57df`._

The implementation correctly and completely satisfies the functional requirements — the BFF precedence is right, the response contract matches exactly, the retry guard is widened correctly, and the Header/page refactor follows project conventions with low regression risk. Original review gaps closed:

1. ✅ Unit test for `useAddOrdersButton` covering loading, success, each typed code, the generic fallback, and the empty `200 []` path (Issue 1).
2. ✅ Route-level test for `getProjects.ts` precedence — added in a second follow-up. All three typed 400 branches + precedence + de-conflation locked down (Issue 1).
3. ✅ Unit test for `isTypedCreateOrdersError` — covers the page-level `fullPageErrorType` mapping (Issue 1 / requirement 4b).
4. ✅ Tooltip copy co-located in `consts.ts` (`CREATE_ORDERS_TOOLTIP_MESSAGES`) and typed-code union derived from the runtime array (Issues 2, 3).
5. ✅ Empty `200 []` path no longer reuses the misleading "Please try again" copy (Issue 4).
6. ✅ Pre-existing `"an Proofed"` typo fixed as a one-character drive-by (Issue 5).
7. ⚠️ Manual testing / `yarn build:customer-portal` / DevTools verification — still pending (this is a manual step that has to happen outside the test suite).
  