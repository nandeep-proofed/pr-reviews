# PR Review: fix/PP-1955: Require Platform, Delivery & Workflow on Order Setup step

**PR:** https://github.com/Proofed/B2BWebserver/pull/2370
**Jira:** https://proofed.atlassian.net/browse/PP-1955
**Status:** Code Review — **all 5 issues triaged, verification pass complete**

---

## Review Outcomes

| # | Issue | Severity | Status | Resolution |
|---|---|---|---|---|
| 1 | PR description claims a `CreateOrderContent` change not in the diff | medium → **low** | ✅ **Fixed** | PR description rewritten to match the actual 4-file diff; added a "How the gate works" section documenting the real mechanism. Docs-only defect — no code change. |
| 2 | `deliveryConfigurationId` type change — ripple not verified | medium → **none** | ✅ **Resolved** | Verified: `typecheck` ✅ and `build` ✅ both pass. Premise was correct (type did change `number \| undefined` → `number`), but no consumer breaks. No code change needed, exactly as predicted. |
| 3 | Graceful submit-time failure not addressed | low → **invalid** | ❌ **Rejected** | Misdiagnosis inherited from the ticket. Graceful failure **already exists** and predates this PR. The endless spinner had a different cause entirely. See below. |
| 4 | Tests cover the schema only, not the UI wiring | low | ⏭️ **Skipped** | Gap is real (the 3 new `setFieldError` lines are untested). Deliberately deferred — tracked as a follow-up. Not blocking a Medium-priority, manually-verified fix. |
| 5 | `type="button"` is out of stated PP-1955 scope | low | ✅ **Resolved** | No change needed. Now mentioned in the PR description. Reframed: it *completes* an existing pattern rather than being scope creep — see below. |

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Require a **Workflow** before letting the user advance — block progression and prompt for the missing selection | `deliveryConfigurationId` is now `Yup.number().min(1).required("Please select a Workflow")` in `ORDER_SETUP_STEP_SCHEMA`; step validation fails and the field-level prompt renders (`showErrorOnTouched` removed from the Workflow select) | ✅ Addressed |
| The user should **never be left on an endless spinner** | Requiring a Workflow guarantees `deliveryConfigurationId` is set, which keeps the `useSelectedDeliveryConfiguration` query **enabled** — this is the actual root cause of the hang (see Issue 3) | ✅ Addressed (true root cause) |
| "If a missing Workflow is only detected at submit, order creation should **fail gracefully with a clear message** and return the user to correct it" | **Already satisfied by pre-existing code** — the submit handler catches per-order failures, routes to `ORDER_FAILED_STEP`, and fires `showDefaultErrorToast`. No change required (see Issue 3) | ✅ Already met |
| (Implied) Same silent dead-end must not occur for Platform / Delivery | Platform now carries an explicit prompt message; **Delivery is newly validated** (`required`) — previously not in the schema at all | ✅ Addressed |

---

## Architecture Analysis

The wizard is a single Formik form (`NewOrderForm/index.tsx`) whose `validationSchema` is swapped per step via `getCreateOrderSchema(step)`. The Order Setup step uses `ORDER_SETUP_STEP_SCHEMA`. The bug was that this schema only required `organizationGroupId` + `platform`, so a blank Workflow (`deliveryConfigurationId`) left the form "valid enough" to advance.

The fix is a clean, root-cause change at the right layer — the step schema — plus two supporting UI tweaks:

1. **`schemas.ts`** — adds `delivery` + `deliveryConfigurationId` as required (and a message on `platform`). `getCreateOrderSchema(ORDER_SETUP_STEP)` is the only runtime consumer that gates this step, so the new rules apply exactly where intended and don't leak into later steps.
2. **`SetupForm/index.tsx`** — removes `showErrorOnTouched` on the three selects so the required error renders once validation populates it (these fields are never in `initialValues`, so they're never marked `touched`, which is why the old gate hid the error). Adds `setFieldError(field, undefined)` inside the three auto-select effects so a field that auto-selects its single option clears any stale error.
3. **`SearchCustomerResultItem.tsx`** — adds `type="button"` to the customer-row button. See Issue 5.

**Behavioural mechanism (differs from the original PR description — now corrected).** On arrival at Order Setup, `formik.errors` is empty (the previous step passed), so with `isInitialValid: false` + a dirty form, `isValid` is `true` and the Next button is **already enabled**. The first Next click (`type="submit"`) runs the step validation, which fails, populates the errors, and blocks progression — and the prompts render because `showErrorOnTouched` was removed. The client's Expected Result is met through the *existing* button gating, not a change to it.

---

## Issues Found

### 1. PR description claims a change to `CreateOrderContent` that is not in the PR — ✅ FIXED

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/CreateOrderContent/index.tsx]**

**Severity:** ~~medium~~ → **low** (documentation defect, zero functional impact)

**Problem:** The PR body stated: *"`.../partials/CreateOrderContent/index.tsx` — Next button enabled (gated only on `isValidating`)…"*. This file is **not** in the PR diff, and on the head commit (`626f0a3`) the button is still `disabled={formik.isValidating || !formik.isValid}` (`CreateOrderContent/index.tsx:95`).

**Verification:** Confirmed. `git diff --name-only develop...626f0a375` returns exactly 4 files, none of them `CreateOrderContent`.

**Impact:** Not a functional bug. The fix works via the pre-existing gating plus the empty-errors-on-arrival invariant. Additionally verified there is **no dead-end** after a failed click: `FormikSelect`'s `onChange` calls `setError(undefined)` then `submitForm()` (`packages/shared/components/atoms/Fields/Select/index.tsx:447`), so selecting an option clears the error and re-enables Next.

**Resolution:** ✅ PR description rewritten — false bullet removed, actual 4-file diff listed, and a new "How the gate works" section documents the real mechanism plus the implicit invariant (errors empty on arrival). The optional one-line button change was **deliberately not made** — unnecessary and would be an untested behaviour change.

### 2. Type change to `deliveryConfigurationId` / new `delivery` field — ✅ RESOLVED (verified clean)

**[File: apps/creative-portal/components/organisms/NewOrderForm/schemas.ts]**

**Severity:** ~~medium~~ → **none** (verified; no action)

**Problem:** `orderSchema` concats `ORDER_SETUP_STEP_SCHEMA` (`schemas.ts:351`), promoting `deliveryConfigurationId` to required in the inferred `CreateOrderSchema` and adding a required `delivery`.

**Verification:** Premise confirmed by probing the resolved type directly — assigning `deliveryConfigurationId` to `string` errors with *"Type 'number' is not assignable to type 'string'"*, proving it is now `number`, not `number | undefined`. Gates run against the PR branch:

- `npx turbo run typecheck --filter=@proofed/creative-portal` → ✅ **pass**
- `npx turbo run build --filter=@proofed/creative-portal` → ✅ **pass**

**Correction to the original reasoning:** this report argued the risk was low because "the base branch already reads `formik.values.delivery` with no `@ts-ignore`, implying the concatenated type is already permissive." That mechanism is wrong. `delivery` did not exist in the schema on develop *at all* — Yup resolves unknown props to `never`, and `never` is assignable to anything, so `values.delivery` silently typed as `never` and compiled. This PR makes it `string`, which **improves** type safety at that call site. Right conclusion, wrong reason.

**Caveat (pre-existing, not introduced here):** `initialValues` at `NewOrderForm/index.tsx:70` carries `// @ts-ignore TEST`, which suppresses exactly the breakage class this issue feared (object literals missing now-required fields). The green typecheck is partly *because* the one constructor is ts-ignored. Defensible — the wizard populates values incrementally — but worth knowing.

**Resolution:** ✅ No code change needed, as this issue itself predicted.

### 3. Secondary Expected Result (graceful submit-time failure) — ❌ REJECTED (invalid: misdiagnosis)

**[File: apps/creative-portal/services/orders/createOrderNew/index.ts]**

**Severity:** ~~low~~ → **invalid**

**What this issue got right:** `onError: () => {}` is a literal no-op (`createOrderNew/index.ts:21`), `?? 0` sits at `api/orders/createNew/utils.ts:206` and `:223`, neither file is in the diff, and the ticket's AC does ask for graceful submit-time failure.

**What it got wrong — the impact claim.** It asserted that "any *other* create failure at that submit path (network error, server 4xx/5xx…) would still spin indefinitely with no toast." **False.** The submit handler already does precisely what this issue proposed as its own fix:

```
NewOrderForm/index.tsx:195   } catch (error) { failedOrdersErrors.push({ id: order.id, error }) }
NewOrderForm/index.tsx:227     actions.setFieldValue("step", ORDER_FAILED_STEP)
NewOrderForm/index.tsx:233     failedOrdersErrors.forEach(f => showDefaultErrorToast(f.error))
```

The no-op `onError` swallows nothing. In React Query, `onError` is a side-effect callback — `mutateAsync` rejects regardless, so the rejection propagates to that `try/catch`. Both paths are covered; the stream mutation (`stream.ts`) has no no-op handler at all and forwards options through. **The AC's second half is already implemented and predates this PR.**

**Where the endless spinner actually came from — not the submit path:**

```
useSelectedDeliveryConfiguration.ts:  enabled: !!organizationGroupId && !!deliveryConfigurationId
PaySummaryStep/index.tsx:31:          if (isLoadingSelectedDeliveryConfiguration) return <Loader />
```

With no Workflow, `deliveryConfigurationId` is undefined → the query is **disabled** → and on React Query **v4.36.1** a disabled query that never fetched reports `status: "loading"` / `isLoading: true` permanently (`fetchStatus: "idle"`). So `PaySummaryStep` renders `<Loader />` forever. Same pattern at `CreateOrderContent/partials/Steps/index.tsx:129,139`. The user never reached the create request, so `onError` and `?? 0` were never involved.

The ticket's own "Technical findings" bullet — *"the mutation's error handler is a no-op `onError` … so the UI remains on the loading spinner"* — is incorrect, and this report repeated it without checking the submit handler. **The PR's fix is better-targeted than either document credits:** requiring `deliveryConfigurationId` guarantees the query is always enabled, killing the hang at its actual source.

**Resolution:** ❌ Rejected — nothing to defer, no follow-up needed. Recommend correcting the ticket's technical-findings bullet so the next person doesn't chase `onError` again. The no-op `onError` is harmless dead code.

**Genuine latent issue surfaced while checking this (separate concern, not this PR's):** the disabled-query-spins-forever pattern is generic. If `organizationGroupId` is ever falsy at `PaySummaryStep`, the same infinite `<Loader />` occurs, and this PR's gate does not cover that. Real defense-in-depth is to distinguish "disabled" from "loading" at those call sites (check `fetchStatus`, or gate on `isInitialLoading` rather than `isLoading`). Worth its own ticket.

### 4. Tests cover the schema only — not the UI wiring the fix depends on — ⏭️ SKIPPED (follow-up)

**[File: apps/creative-portal/components/organisms/NewOrderForm/schemas.test.ts]**

**Severity:** low

**Problem:** The 10 new tests (verified: `10 passed`) correctly assert the schema rules and exact messages, but are schema-only. Confirmed there is **no** test anywhere touching `SetupForm` or `OrderSetupStep`.

**Refinement:** of the three items originally listed as untested, only one is actually *new code*:

- **Next button gating** — not new; this PR does not touch it (see Issue 1). Pre-existing untested behaviour.
- **`showErrorOnTouched` removal** — a deletion; observable only through a render.
- **The three `setFieldError(<field>, undefined)` lines** — genuinely new code in `626f0a375`, zero coverage. This is the piece that trips the CLAUDE.md "all new code must have tests" rule, and the one most worth protecting: it exists only because auto-select uses `setFieldValue`, which (unlike the select's own `onChange`) doesn't clear an existing error. Nothing in the code says that out loud — a future reader could delete it as redundant with every test still green.

**Obstacle:** `SetupForm/` has no `hooks.ts` — 10 hook calls and 3 data queries sit directly in `index.tsx`, violating the "`index.tsx` is UI-only" convention. That makes a full RTL test expensive (mock 3 query modules + Formik context + drive react-select). The cheaper, convention-aligned path is to extract the auto-select effects into `hooks.ts` and cover them with `renderHook` — matching `BriefStep/hooks.test.ts` and `WorkflowStep/hooks/useDeadlineManagement.test.ts` in the same folder tree.

**Resolution:** ⏭️ **Skipped by decision.** Deferred to keep this PR to its 4-file diff and avoid a refactor inside a fix PR. Tracked as a follow-up covering: extract `SetupForm` effects to `hooks.ts`, add `renderHook` coverage for the auto-select error-clearing, and add a behavioural test for the prompt/block. Not blocking — the fix is manually verified on `b2btest.proofed.com` and the schema rules themselves are well covered.

### 5. `type="button"` fix is out of the stated PP-1955 scope — ✅ RESOLVED (no change needed)

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/CustomerSearch/partials/SearchCustomerResultItem.tsx]**

**Severity:** low (observation)

**Verification:** The fix is correct and necessary. `ResultsListItemRowButton` is `styled(RawButton)`, and `RawButton` → `StyledRawButton = styled(ThemedButton)` (theme-ui `Button`) renders a native `<button>` with **no** explicit `type`, which defaults to `type="submit"` inside the wizard's `<Form>` (`NewOrderForm/index.tsx:376`).

**Reframed:** this is **not** scope creep. The sibling `ResultsListItemActionButton` **already carries `type="button"` on develop** (`SearchCustomerResultGroup.tsx:52`), so the codebase had already established this pattern and `SearchCustomerResultItem` was simply the instance that was missed. This PR *completes* the pattern. (The third `styled(RawButton)` — `ResultsFooterButton` — has no usages; dead code, irrelevant.)

**Resolution:** ✅ No code change needed. Now documented in the PR description's "Areas of Change" — the only action this issue asked for.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run typecheck --filter=@proofed/creative-portal` | ✅ Pass | Resolves Issue 2 — the type ripple is clean |
| `npx turbo run build --filter=@proofed/creative-portal` | ✅ Pass | 0 type warnings |
| `npx vitest run schemas.test.ts` | ✅ Pass | 10/10 tests |
| `npx turbo run lint --filter=@proofed/creative-portal` | ⚠️ 5 errors | **Unrelated / pre-existing.** All 5 are `prettier/prettier` formatting in `components/molecules/JobReturnTimesTray/index.test.tsx` — untouched by this PR, last modified by PP-1953 (#2359, already merged). Per PR-scope discipline, left alone. **develop is currently red on lint** — worth a separate cleanup. |

---

## Tests

- ✅ New unit tests (`schemas.test.ts`, 10 tests) — verified passing. Cover the three required selections, exact prompt messages, the `id === 0` invalid case, and the `getCreateOrderSchema(ORDER_SETUP_STEP)` gate.
- ⏭️ No component/behavioural test for the UI wiring — **skipped by decision**, tracked as follow-up (Issue 4).
- ✅ Typecheck + build verified clean on the PR branch (Issue 2 resolved).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Root cause fixed at the right layer; both halves of the AC are met (one by this PR, one by pre-existing code) |
| Regression risk | ✅ **Low** — type ripple verified clean via typecheck + build (was "medium, unverified") |
| Tests | ⚠️ Schema tests solid and passing; UI-behaviour gap knowingly deferred |
| Code quality | ✅ Clean, idiomatic, well-scoped |
| Validation suite | ✅ Run — typecheck/build/tests green; lint red only on a pre-existing unrelated file |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve.** (Upgraded from "Approve with suggestions" — the two conditions have been discharged.)

1. ~~Run the validation suite before merging~~ → ✅ **Done.** typecheck ✅, build ✅, tests ✅. Lint's 5 errors are pre-existing and unrelated (`JobReturnTimesTray/index.test.tsx`, from #2359).
2. ~~Fix the PR description~~ → ✅ **Done.** Rewritten to the actual 4-file diff with an accurate mechanism section.
3. **Add a component-level test** (Issue 4) → ⏭️ **Skipped by decision**, follow-up ticket to be raised.
4. ~~Note the deferred half of the AC~~ (Issue 3) → ❌ **Withdrawn** — the graceful-failure path already exists; the original concern was a misdiagnosis inherited from the ticket.
5. ~~Mention the `type="button"` fix~~ (Issue 5) → ✅ **Done** in the PR description.

**Post-merge follow-ups worth raising separately:**

- **Disabled-query spinner (from Issue 3):** `isLoading` is `true` forever for a disabled React Query v4 query. `PaySummaryStep` and `Steps` render `<Loader />` on it. This PR closes the Workflow path, but a falsy `organizationGroupId` would reproduce the same infinite spinner. Gate on `isInitialLoading` / `fetchStatus` instead.
- **`SetupForm` hooks extraction + tests (from Issue 4).**
- **develop lint is red** — `JobReturnTimesTray/index.test.tsx`, 5 auto-fixable prettier errors.
- **Ticket correction:** PP-1955's "Technical findings" bullet blaming the no-op `onError` for the spinner is incorrect and should be amended.

The core change is correct and well-targeted — arguably more so than the original review credited, since it fixes the true root cause (a permanently-disabled query) rather than the one the ticket hypothesised.
