# PR Review: feature/PP-1890: DivisionalHierarchy migration (classification selector, flag-based gating, removable stub)

**PR:** https://github.com/Proofed/B2BWebserver/pull/2340 (draft, 127 files, +2685 / −758, 28 commits)
**Jira:** https://proofed.atlassian.net/browse/PP-1890
**Status:** Code Review
**Reviewed at commit:** `df2b2cb21bf478e655aac139d7a4843137f1026d`

> **Note on requirements:** the Jira description was amended twice in the comments. The 2026-06-24 comment struck out requirement 3.2 (deriving ServiceDelivery/ServiceSupport into `roles`), and the 2026-06-30 "New Permissions Model — Decisions" comment set the final gating targets. This review treats the comments as authoritative over the original description.
>
> **Note on the BE contract (PP-1934):** two later decisions on [PP-1934](https://proofed.atlassian.net/browse/PP-1934) override §8.2/§8.3 as written in the PP-1890 description, and are load-bearing for Issues 2, 19 and 25:
>
> 1. **2026-06-17 (Hideshi)** — `divisionalHierarchyId` is **rejected** on User Search and Lookup ("strictly internal security data"). User Search returns `classification`; User Lookup gains `classification` plus `userAdmin` / `orderAdmin` / `orgAdmin`. This validates the PR's classification-string matching approach (Req 1.6).
> 2. **2026-07-03 (Hideshi)** — `divisionalHierarchyId` on User Update is now **conditional**: _"It is only required when the requester id differs from the target user… self-updaters cannot specify the divisionalHierarchyId."_ Self-edit must therefore **omit** the field; other-edit must **send** it.
>
> Point 2 is why Issue 2's fix is conditional rather than an unconditional `.required()` — see that issue.

---

## Jira Requirements vs Implementation

| Jira Requirement                                            | PR Implementation                                                                                                                                                           | Status                  |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| 1.1 Classification radio group on Create + Edit             | `ClassificationRadioGroup` partial, rendered in both modes                                                                                                                  | ✅ Addressed            |
| 1.2 System Admin not shown as an option                     | `isAssignableClassification` filters `"System Administration"` — but by hardcoded English string (Issue 8)                                                                  | ⚠️ Partial              |
| 1.3 Options from Hierarchy Search §4.1                      | New `GET /api/divisionalHierarchy` + `useDivisionalHierarchyQuery`                                                                                                          | ✅ Addressed            |
| 1.4 Classification required on **both** Create and Edit     | Required on Create only; `buildEditTeamMemberSchema` omits the rule. Per PP-1934 this is correct for _self_-edit but wrong for other-edit (Issue 2)                         | ❌ Missing (other-edit) |
| 1.5 Trust BE-filtered options                               | Options rendered as returned                                                                                                                                                | ✅ Addressed            |
| 1.6 Edit pre-selects by matching `classification` string    | `matchedHierarchyId` in `useTeamMemberModal`                                                                                                                                | ✅ Addressed            |
| 2.1 `divisionalHierarchyId` on Create + Update              | Sent on both; omitted on Edit when unmatched. PP-1934 made Update conditional (self-edit must omit) — the code gets self-edit right by accident, other-edit wrong (Issue 2) | ⚠️ Partial              |
| 2.2 Legacy `roles` still sent                               | `buildRolesForSubmit` keeps visible roles                                                                                                                                   | ✅ Addressed            |
| 3.1 SD/SS toggles not rendered                              | Removed from the form                                                                                                                                                       | ✅ Addressed            |
| 3.2 ~~Derive SD/SS into roles~~ (**struck out** 2026-06-24) | Correctly _not_ derived — `buildRolesForSubmit` is subtractive only                                                                                                         | ✅ Correct              |
| 3.3 Editor/Reviewer/Returner/QA remain editable             | Unchanged                                                                                                                                                                   | ✅ Addressed            |
| 4.2a Team-management → `userAdmin`                          | `flagGuard("userAdmin")`, `MemberActions`, table                                                                                                                            | ✅ Addressed            |
| 4.2b Org/partner/customer → `organizationAdmin`             | `flagGuard("organizationAdmin")`, `useHasPartnerAccess`, `CustomerTable` — except `MainLayout` (Issue 3)                                                                    | ⚠️ Partial              |
| 4.2c Reporting → `organizationAdmin` (per 06-30 decision)   | `flagGuard("organizationAdmin")` on the reporting page                                                                                                                      | ✅ Addressed            |
| "Keep a **None** option" (06-30 decision)                   | Implemented as a _relabel_ of the BE `Creative` classification, not a null option — correct, since the id must be numeric                                                   | ✅ Addressed            |
| "Remove Service Delivery Head" (06-30 decision)             | Filtered via `HIDDEN_CLASSIFICATIONS`                                                                                                                                       | ✅ Addressed            |
| "Returner-only users keep `isAdministrative = true`"        | Explicit `Returner` clause in `isAdministrative`                                                                                                                            | ✅ Addressed            |
| "**No change** to Settings & Onboarding gating"             | Semantics changed: SD/SS lose onboarding steps, plain `Admin` gains settings tiles (Issue 6)                                                                                | ❌ Deviates             |
| "**No change** to Availability role checks"                 | Switched to `flagGuard("userAdmin")` (Issue 7)                                                                                                                              | ❌ Deviates             |
| "**No change** to the shared administrative-access helper"  | `isAdministrative` rewritten, signature + semantics changed (Issue 1 context)                                                                                               | ⚠️ Deviates             |
| 5. Surface auth flags in user context                       | Flags reach the session via object spread in `packages/shared/api/login`                                                                                                    | ⚠️ Partial (Issue 4)    |
| 6. Out of scope: Order Management UI unchanged              | PR adds ~220 lines of Order-sidebar "Created by" work (Issue 13)                                                                                                            | ❌ Scope creep          |

---

## Architecture Analysis

The overall shape is sound and the abstractions are the right ones. `resolveAdminFlags` centralises flag resolution, `useAdminFlags` wraps it for the client, and `flagGuard` / `administrativeGuard` give a clean, declarative SSR gate that replaces scattered role arrays. The new API route follows the existing `masterReferences` pattern faithfully (`withApiMiddleware` + `handleEndpointError` + `makeMethodToHandlerMapping` + thin `pages/api` re-export), and `useDivisionalHierarchyQuery` correctly sets `retry: false` because a plain requester gets a 401. Thirteen new/updated test files all pass.

The problems are concentrated in two places:

1. **A circular dependency between two new pieces of logic.** `resolveAdminFlags` deliberately treats the legacy `Admin`/`Superadmin` roles as the safety net that prevents a System Administrator from ever being locked out. `buildRolesForSubmit` deletes exactly those roles from the payload. The two were written against opposite assumptions, and the collision is unrecoverable through the UI (Issue 1).

2. **Divergence from the 2026-06-30 decisions.** That comment drew a careful line — flags replace role checks _for page gating_, but Settings, Onboarding, Availability and the shared helper stay on roles. The PR crossed that line in all four places. Each individually looks like a tidy refactor; collectively they change who can see what, and none of the changed surfaces has a test asserting the _old_ behaviour is preserved.

---

## Issues Found

### 1. Editing any System Administrator permanently locks them out

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/utils.ts]**

**Function/Class:** `ADMIN_CLASS_ROLES` / `buildRolesForSubmit` (lines 22–35), applied at `hooks.ts:127`

**Severity:** high

**Problem:** `buildRolesForSubmit` strips `Admin` and `Superadmin` from the submitted `roles` array, in addition to `ServiceDelivery`/`ServiceSupport`:

```ts
export const ADMIN_CLASS_ROLES = new Set<UserRole>([
  UserRole.ServiceDelivery,
  UserRole.ServiceSupport,
  UserRole.Admin,
  UserRole.Superadmin
]);
```

But `utils/resolveAdminFlags.ts:20-28` states the exact opposite dependency in its own comment:

```ts
// Until the BE confirms whether System Administrators receive the admin flags
// (the DivisionalHierarchy lists no flags for the System Administration
// division), the legacy Admin/Superadmin roles still grant full admin access so
// a system admin is never locked out (PP-1890).
const SYSTEM_ADMIN_ROLES = new Set<UserRole>([
  UserRole.Admin,
  UserRole.Superadmin
]);
```

The ticket only ever asked for `ServiceDelivery`/`ServiceSupport` to stop being sent. On `develop`, `Admin`/`Superadmin` were never form checkboxes but _were_ preserved in the PUT via `{ ...editedUser, ...values }`.

**Impact:** A Superadmin is edited via the Team Member modal — even a no-op save of a phone number. `Superadmin` is stripped from `roles`. Because the System Administration division carries no admin flags, `resolveAdminFlags` now returns all-false and `isAdministrative` returns false. On next login that user is bounced to `/no-access` on every admin page. They cannot restore their own access (Team Members is gated on `userAdmin`), and **no other admin can restore it either**, because `HIDDEN_DIVISIONS` removes System Administration from the radio options. This is irreversible from the UI and requires a direct BE/DB fix. `__tests__/hooks.test.ts:227-248` asserts `["superadmin","returner"] → ["returner"]`, so the behaviour is deliberate but, I believe, unintended in consequence.

**Fix:** Restrict the submit-path filter to the two roles the ticket named, and keep `Admin`/`Superadmin` flowing through untouched:

```ts
// Roles hidden from the form and never re-sent (PP-1890 Req 3.1). Admin and
// Superadmin are deliberately NOT included: resolveAdminFlags relies on them as
// the System Administrator fallback until Auth §5.1 flags are confirmed.
export const ADMIN_CLASS_ROLES = new Set<UserRole>([
  UserRole.ServiceDelivery,
  UserRole.ServiceSupport
]);
```

If a separate display-side filter genuinely needs to hide `Admin`/`Superadmin` chips, use a second constant for that and keep it off the submit path. Update `__tests__/hooks.test.ts:227-248` to assert `superadmin` is **preserved**.

---

### 2. Classification is not required on Edit, contradicting Req 1.4

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/partials/Form/consts.tsx]**

**Function/Class:** `buildEditTeamMemberSchema` (lines 121–128) vs `buildNewTeamMemberSchema` (lines 101–110)

**Severity:** high

**Problem:** The create schema requires the field; the edit schema has no `divisionalHierarchyId` rule at all. The roles validator also short-circuits on an empty classification (`consts.tsx:58`: `if (!divisionalHierarchyId) return true;`).

The stated rationale — preserve a member holding a non-grantable classification like Service Delivery Head — is reasonable, but the implementation is far broader than the rationale. `hooks.ts:89-90` sets `resolvedHierarchyId = ""` whenever `matchedHierarchyId` is null, which also happens when the hierarchy query 401s, fails, or has not yet resolved. `hooks.ts:128-130` then omits `divisionalHierarchyId` from the PUT entirely.

**Impact — refined by PP-1934 (2026-07-03).** The BE made `divisionalHierarchyId` **conditional** on Update: required only when the requester differs from the target, and self-updaters _cannot_ specify it. That splits this issue in two:

- **Self-edit — currently correct, but by accident.** §4.1 returns only classifications _below_ the requester, so a requester's own classification never matches, `resolvedHierarchyId` is `""`, and the field is dropped — exactly what the BE now requires. Nothing in the code expresses this intent: `hooks.ts:117-120` attributes the omission to "a non-grantable classification", and `isSelfEdit` is computed at `hooks.ts:69` but only ever used to disable the radio. A future change to §4.1's contents would silently break self-update with no failing test.
- **Other-edit — still broken.** `hooks.ts:89-90` sets `resolvedHierarchyId = ""` whenever `matchedHierarchyId` is null, which also happens when the hierarchy query 401s, fails, or has not yet resolved — and when the member has **no classification at all**, since `canManageMember` deliberately returns `true` in that case (`TeamMembersTable/utils.ts:17-19`, "never blank the menu"). So Edit is offered for an unmigrated member, the form validates clean, and the PUT omits a property the BE requires whenever requester ≠ target → **400**. Separately, an admin can still save an Edit with no classification _and_ zero roles, producing an unclassified, permission-less user — a **new** hole, not a latent one: before this PR `roles` carried an unconditional `Yup.array().min(1, "At least 1 role must be selected.")`, so a zero-role save was impossible. `consts.test.ts:75-84` locks the current behaviour in as correct.

**Fix:** Branch on `isSelfEdit` — do **not** add an unconditional `.required()`, which would break self-update and reintroduce the exact bug PP-1934 was raised to fix.

```ts
export const buildEditTeamMemberSchema = (
  nonAdminHierarchyIds: string[] = [],
  isSelfEdit = false
) =>
  Yup.object().shape({
    ...buildCommonSchema(nonAdminHierarchyIds),
    ...LegacyIdsSchema,
    phone: Yup.string().required("Phone number required."),
    // PP-1934: BE requires the id only when requester !== target;
    // self-updaters must not send it at all.
    ...(isSelfEdit
      ? {}
      : {
          divisionalHierarchyId: Yup.string().required(
            "Classification required."
          )
        })
  });
```

Pass `isSelfEdit` from `hooks.ts:375`, and make the submit path drop the field on `isSelfEdit` explicitly rather than inferring it from an empty string. For the genuinely non-grantable other-edit case (Service Delivery Head), use an explicit sentinel (`KEEP_CURRENT_CLASSIFICATION`) set only when the classification was **found** in the hierarchy but is non-assignable — never when the lookup merely failed — and strip it at submit time. Update `consts.test.ts:75-84`, and add a test asserting self-edit omits the field while other-edit sends it.

---

### 3. `MainLayout` reads the raw flag, bypassing the System Admin fallback

**[File: apps/creative-portal/components/layouts/MainLayout/hooks.ts]**

**Function/Class:** `useMainLayout`, lines 50–52

**Severity:** high

**Problem:** This is the only organization-gated surface that reads the session field directly instead of going through `resolveAdminFlags`:

```ts
const creationDropdown = isAdmin
  ? getCreationDropdown(!!user.organizationAdmin)
  : undefined;
```

A repo-wide grep for raw `user.organizationAdmin` / `user.userAdmin` / `user.orderAdmin` returns **exactly one production hit** — this line. Every sibling uses the resolver: `useHasPartnerAccess.ts:7`, `CustomerTable/hooks/useCustomerTable.ts:27`, `partners/[partnerId]/projects/[projectId]/settings/hooks.ts:344`, and `flagGuard("organizationAdmin")` on the partners page.

**Impact:** A legacy Admin/Superadmin with no `organizationAdmin` flag sees no "Partner" entry in the creation dropdown, yet `flagGuard("organizationAdmin")` grants them `/partners` and `useHasPartnerAccess` returns true. The UI and the guard disagree, and the documented no-lockout guarantee is silently broken in this one spot.

**Fix:**

```ts
const { organizationAdmin } = useAdminFlags();

const creationDropdown = isAdmin
  ? getCreationDropdown(organizationAdmin)
  : undefined;
```

---

### 4. A flag-less auth response causes a silent mass lockout with no observability

**[File: apps/creative-portal/utils/resolveAdminFlags.ts]**

**Function/Class:** `resolveAdminFlags`, lines 41–45

**Severity:** medium

**Problem:** `undefined` coerces to `false`, so the resolver fails closed. That is the correct default (no privilege escalation), but there is no log, warning, or metric when it happens. The flags reach the session only by object spread — `packages/shared/api/login/index.ts` does `req.session.user = data as any` with no explicit mapping — so a contract drift (for example the response returning `orgAdmin` instead of `organizationAdmin`, which the PR description itself flags as unverified) produces zero type errors and zero runtime signal.

**Impact:** A BE deploy-ordering issue or a per-user data gap locks out every ServiceDelivery/ServiceSupport user simultaneously, and the failure is indistinguishable from correct behaviour in logs.

**Fix:** Map the three flags explicitly rather than spreading, and emit a warning at login when a user holds a legacy SD/SS role but has all three flags absent:

```ts
if (
  !userAdmin &&
  !orderAdmin &&
  !organizationAdmin &&
  roles?.some((role) => LEGACY_ADMIN_ROLES.has(role))
) {
  logger.warn("PP-1890: legacy admin user received no admin flags", {
    userId
  });
}
```

Confirm the exact property names against the deployed §5.1 response before merging — this is already listed as an open item in the PR description.

---

### 5. The main nav advertises pages the new guards reject

**[File: apps/creative-portal/components/molecules/MainNav/consts.ts]**

**Function/Class:** `adminMainNavLinks` (lines 25–63); `MainLayout/hooks.ts:48` computes `isAdmin`, which drives `areaType` at :127, and the list is selected at :64–65. The file is unchanged by this PR

**Severity:** medium

**Problem:** The nav list is static — Orders, Partners, Team Members, Customers, Availability, Reporting — but the destinations are now gated by three different flags.

**Impact:** A Returner is `isAdministrative` by design, but has all flags false, so Partners / Team Members / Customers / Availability / Reporting all bounce to `/no-access`. Likewise a `userAdmin`-only admin clicking Customers or Reporting, and an `organizationAdmin`-only admin clicking Team Members. The gap existed for Returner before, but the flag split materially widens it — this is the most visible day-one symptom for real users.

**Fix:** Attach the required `AdminFlag` to each nav entry and filter with `useAdminFlags()`:

```ts
const flags = useAdminFlags();
const visibleLinks = adminMainNavLinks.filter(
  (link) => !link.requiredFlag || flags[link.requiredFlag]
);
```

---

### 6. Settings & Onboarding gating semantics changed, despite the "no change" decision

**[File: apps/creative-portal/components/pages/onboarding/const.tsx]**

**Function/Class:** `ADMIN_STEP_VISIBILITY` (lines 12–18); and `components/pages/settings/consts.tsx:24-30` `PERSONAL_VISIBILITY`

**Severity:** medium

**Problem:** These read as refactors but are not behaviour-preserving.

_Onboarding:_ `ADMIN_STEP_VISIBILITY` replaces explicit lists that included `Returner, Admin, Superadmin, ServiceDelivery, ServiceSupport`. `forAdmin` resolves via `isAdministrative`, which no longer matches SD/SS **by role**.

**Correction:** this is a transition-window risk, not a flat regression. Under a properly-migrated Rev 4.0 backend, former SD/SS users hold classifications 1003–1005, which carry `orderAdmin`/`organizationAdmin` — so `isAdministrative` returns true and `forAdmin` covers them. `Returner` is matched explicitly and `Admin`/`Superadmin` via the system-admin fallback. Steps are lost **only** for a user who still carries SD/SS in `roles` while their flags are absent — i.e. a pre-Rev-4.0 response or an incomplete BE migration.

_Settings:_ the previous lists were `[Editor, Reviewer, Returner, Superadmin, ServiceDelivery, ServiceSupport]` — note `Admin` was absent. `showToAdmin` → `isAdministrative` now includes `Admin`, so plain `Admin` users **newly gain** Personal Information, Password, Two-Factor Authentication, Payment, Languages, Your profile and Degrees tiles.

**Impact:** Two unrequested access changes on surfaces the 2026-06-30 decision explicitly fenced off. Note the settings half cuts **both** ways: the old lists also named `ServiceDelivery`/`ServiceSupport` explicitly, so an unmigrated SD/SS user loses Personal Information, Password, 2FA, Payment, Languages, Your profile and Degrees — the same transition-window loss flagged for onboarding, not only the `Admin` gain. The settings change is probably a desirable fix, but it is untested and unsanctioned. The onboarding change is safe on a fully-migrated backend and only bites during rollout — but nothing pins that, and the PR description itself lists the flag contract as unverified against the deployed service.

**Fix:** Do **not** re-add `ServiceDelivery`/`ServiceSupport` to the roles list — Orlin confirmed on 2026-06-30 that those values are _"no longer returned in the `roles` array"_ under Rev 4.0, so entries for them would be inert and would misrepresent the model as still role-driven.

If the transition window needs covering, the safety net belongs where the existing one already lives — `resolveAdminFlags`, alongside the `Admin`/`Superadmin` fallback — so every flag consumer benefits, not just onboarding:

```ts
// resolveAdminFlags.ts — mirror of SYSTEM_ADMIN_ROLES, removable once the
// BE migration is confirmed complete in every environment.
const LEGACY_ADMIN_ROLES = new Set<UserRole>([
  UserRole.ServiceDelivery,
  UserRole.ServiceSupport
]);
```

For the settings half, get the decision amended in the ticket. Add tests pinning the resulting behaviour either way.

---

### 7. Availability switched to `userAdmin`, contradicting the 2026-06-30 decision

**[File: apps/creative-portal/components/pages/availability/index.tsx]**

**Function/Class:** `getServerSideProps`, lines 26–28

**Severity:** medium

**Problem:** Now `withUserProvided(flagGuard("userAdmin"))`; previously `[Superadmin, Admin, ServiceDelivery, ServiceSupport]`. The ticket description says availability → `userAdmin`; the later decision comment says "**No change** to … the Availability role checks — roles aren't being removed."

**Impact:** A genuine access change either way — SD/SS users without `userAdmin` lose the page. The two requirement sources disagree, so this needs a decision, not a code fix.

**Fix:** Confirm with Orlin which source wins. If the description stands, no change; if the comment stands, revert to `withUserProvided([...roles])`.

---

### 8. System Admin exclusion depends on exact English display strings and fails open

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/utils.ts]**

**Function/Class:** `isAssignableClassification` (lines 13–17); constants at 10–11

**Severity:** medium

**Problem:**

```ts
const HIDDEN_DIVISIONS = ["System Administration"];
const HIDDEN_CLASSIFICATIONS = ["Service Delivery Head"];
```

A BE rename, casing change, or stray whitespace makes the filter return `true`.

**Impact:** **System Administrator becomes a selectable radio option** — a direct violation of Req 1.2, failing open with no error and no test failure. This is a UI-correctness defect, not an escalation vector: the option only becomes _visible_; the BE still enforces tier rules on Update, which is why the `isTierLevelError` path in Issue 10 exists at all.

**Fix:** Filter on structural data rather than display strings. `DivisionalHierarchyEntry` already carries `tier`, `orderAdmin` and `organizationAdmin` (`api/divisionalHierarchy/types.ts:1-8`), so a tier threshold — or, better, a BE-provided `assignable` flag — is robust. At minimum, normalise case and trim, and report to Sentry when the hierarchy contains a division matching neither list.

---

### 9. `enableReinitialize` discards in-flight user input on Edit

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/hooks.ts]**

**Function/Class:** `useTeamMemberModal`, lines 359–378

**Severity:** medium

**Problem:** `enableReinitialize: !!editedUserId || !!editedUser` is **pre-existing** (line 331 of the base `hooks.ts`), and `initialValues` already depended on the async `editedUser` lookup. What this PR adds is a **second, later-resolving reinitialisation trigger**: `resolvedHierarchyId` enters `initialValues` at :365 and its dependency array at :368, driven by a query that only fires once the modal opens (`enabled: !!isOpen`, :65–67). The PR widens an existing window rather than opening a new one.

**Impact:** On a slow connection, everything typed while the hierarchy request is in flight is silently reset.

**Fix:** Gate the form render on the hierarchy query settling (hoist the existing `LoadingWrapper` above the form), or set the resolved id via `setFieldValue` on first load instead of routing it through `initialValues`.

---

### 10. Tier rejection is detected by exact error-message equality

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/GeneralOrderInfo/utils.ts]**

**Function/Class:** `isTierLevelError`, lines 17–27 (+ `consts.ts:7`)

**Severity:** medium

**Problem:** The check ends `return message === TIER_LEVEL_ERROR_MESSAGE;` (line 26), where the constant holds the literal `"Must be equal/lower than requester's Tier level"`. The HTTP status is available and is the robust signal — `packages/shared/api/utils/handleEndpointError.ts:42-47` explicitly propagates `error.response.status` from the OMS error through to the client.

**Impact:** Any backend copy edit (punctuation, casing, localisation) silently reverts the UI to the broken state — error toast plus a permanently stuck skeleton. `utils.test.ts` asserts against the same constant, so it can never catch the drift.

**Fix:** Key on `error.response.status` as the primary discriminator, keeping the message check as secondary narrowing. Prefer **403** — 401 is also the session-expiry status here, so treating it as a tier rejection would mask an expired session.

---

### 11. Genuine errors still produce a permanently stuck skeleton

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/GeneralOrderInfo/partials/CreatedByContent/index.tsx]**

**Function/Class:** `CreatedByContent`, lines 42–53

**Severity:** medium

**Problem:** On a 500 or network failure, `isTierRestricted` is false and `isCreatorLoading` is false, so the component falls through to `AssigneeView` with `userDetails` undefined — and `AssigneeView/index.tsx:20-22` renders an indefinite `SkeletonBox` when `userDetails` is falsy.

**Impact:** The skeleton only sticks when `createdForId` is absent. When it is present, the second query (enabled whenever `!!createdForId`, `hooks.ts:46-49`) resolves and `AssigneeView` renders **the wrong person's name** — arguably worse, since it is silently incorrect rather than visibly broken. Either way the PR fixes only the tier case. The `creatorDisplayName` fallback is already available and costs nothing.

**Fix:** Fall back on _any_ error, and reserve the toast for the unexpected ones. This also downgrades Issue 10 from a correctness bug to cosmetic noise:

```tsx
if (isError) return <Text>{creatorDisplayName}</Text>;
```

---

### 12. Expected tier rejections retry 3× and are reported to Sentry

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/GeneralOrderInfo/hooks.ts]**

**Function/Class:** creator personal-info query, lines 26–38

**Severity:** medium

**Problem:** Two separate issues in one query. (a) No `retry: false`, so React Query's default of 3 retries with backoff applies to a deterministic, will-never-succeed rejection — roughly 7 seconds of skeleton before the fallback name appears. (b) The local `onError` suppresses the toast, but `packages/shared/utils/createAppQueryClient.ts:32-46` installs a `QueryCache.onError` that calls `reportError` for every query error regardless of per-query options, so the comment claiming the error is "swallowed" is inaccurate.

**Impact:** A slow, visibly-degraded sidebar for below-tier viewers, plus a Sentry exception on every order-sidebar open by such a user.

**Fix:** Add `retry: false` (matching the precedent already set in `services/divisionalHierarchy/index.ts:28`) and add `meta: { skipSentry: true }` honoured in the cache handler.

---

### 13. Order-sidebar work is explicitly out of scope for this ticket

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/GeneralOrderInfo/**]\*\*

**Function/Class:** `GeneralOrderInfo` / `CreatedByContent` (~220 lines across 7 files)

**Severity:** medium

**Problem:** PP-1890 §6 Out of Scope states "Order Management UI is unchanged (OMS Phase 2)." The code itself confirms the mismatch — the surrounding tests are labelled `(PP-1867)` and the comments reference PP-1750 and PP-1867, not PP-1890.

**Impact:** Makes a 127-file PR harder to review and impossible to revert independently. There is a defensible argument that the tier restriction is _surfaced_ by this migration, but the fix belongs in its own ticket.

**Fix:** Split into a separate PR against the PP-1867 lineage, or get the scope amendment recorded on PP-1890.

---

### 14. Convention deviations (CLAUDE.md)

**Severity:** low

Consolidated from the domain review and the file-by-file pass. Grouped by kind.

**Component structure**

- **`.../TeamMemberModal/partials/Descriptions/`** — one of two component folders under `TeamMemberModal` with neither `hooks.ts` nor `types.ts` — it contains just `index.tsx`, and `FormikCheckboxesGroupedRoles/` has only `index.tsx` + `styles.tsx`. `index.tsx:14-26` also violates the "index.tsx is UI-only" rule — the body does data fetching and derivation (`useFormikContext`, `useDivisionalHierarchyQuery`, a `.find()`), and line 14 annotates `React.FC` with no `Props`. The sibling `ClassificationRadioGroup/` does all of this correctly. Extract `useTeamMemberDescriptions` into `hooks.ts` and add `types.ts` in the same pass.
- **`.../TeamMembersTable/partials/RolesFilter/index.tsx:24-43`** — non-trivial branching in an inline `onChange`; belongs in the sibling `hooks.ts`.
- **`.../TeamMemberModal/index.tsx:21`** — `useBlockBodyWhenMenuIsOpened(!!isOpen)` is the one remaining effect in that file; `isOpen` is already passed to `useTeamMemberModal` at line 25, so it can move into the hook.
- **`GeneralOrderInfo/` and `RolesFilter/` have no `types.ts`** — their prop types live two folders up. `CreatedByContent/`, created by this PR, gets it right.
- **`.../TeamMemberModal/partials/Form/consts.tsx`** — `.tsx` extension but contains no JSX (pure Yup schemas and role arrays). Rename to `consts.ts`.

**Types and naming**

- **`.../TeamMemberModal/types.ts:5`** — `onClose: () => void` should be `VoidFunction`. `hooks.ts:47` already does this, so the same callback is declared two ways in one folder.
- **`.../TeamMembersTable/types.ts:82-87`** — `MemberTeamFilterProps` is flat-alphabetical (`filters, options, placeholder?, setFilters`) instead of required-first (`filters, options, setFilters`, then `placeholder?`). Not caught by lint because it is a `type` alias, not an `interface`.
- **`.../TeamMemberModal/index.tsx:16-20`** — destructuring order is `onClose, isOpen, editedUserId`; should be required-first then optional-alphabetical (`onClose, editedUserId, isOpen`).
- **`.../TeamMemberModal/partials/Form/consts.tsx:37, 82`** — `buildCommonSchema` / `LegacyIdsSchema` are both named `…Schema` but neither is a Yup schema (they are field bags spread into `Yup.object().shape()`), and `LegacyIdsSchema` is PascalCase among SCREAMING_SNAKE siblings.
- **`.../RolesFilter/index.tsx:7`** — `React.FC` namespace access with no `React` import (resolves only via the UMD global); siblings touched in this PR use `import { FC } from "react"`.

**Duplication and reuse-first**

- **`.../ClassificationRadioGroup/index.tsx:23, 27`** — `"Administrative Roles"` appears twice, as `aria-label` on the `role="radiogroup"` wrapper and as visible `<Text>`, with nothing keeping them in sync. Prefer `aria-labelledby` pointing at the visible heading, or hoist the string to a sibling `consts.ts`.
- **`.../ClassificationRadioGroup/hooks.ts:36-41, 56-60`** — the `value`/`label` derivation is duplicated verbatim within one `useMemo`. Extract to `utils.ts` next to `isAssignableClassification`, where `utils.test.ts` already provides coverage. Note `hooks.test.ts:26` mocks `CLASSIFICATION_INFO` as `{}`, so the `?? entry.classification` fallback is the only branch exercised at either site.
- **`.../TeamMemberModal/consts.ts:45, 72-74`** — two nits on the same constant: line 72 hardcodes the `"Creative"` key when `NON_ADMIN_CLASSIFICATION` already exists and is imported by three other call sites, and line 45 exports `NONE_CLASSIFICATION_LABEL` which is consumed only in this file (zero external references). Use a computed key and drop the export: `[NON_ADMIN_CLASSIFICATION]: { label: NONE_CLASSIFICATION_LABEL }`.
- **`CreatedByContent/index.tsx:31-40` and `TeamMembersTable/consts.tsx:355-364`** — raw `SkeletonBox` instead of the mandated shared `LoadingWrapper`. `GeneralOrderInfo/index.tsx:122-125` — same PR, same component tree — does it the prescribed way, so this is internal inconsistency rather than an unavoidable exception.
- **`.../Descriptions/index.tsx:31-46`** — `<Box mt="2"><Text variant="text3">` is duplicated in both ternary branches; line 36 is a backtick template literal with no interpolation (CLAUDE.md mandates double quotes); the two long copy strings belong in `consts.ts`.
- **`.../Form/index.tsx:25-38, 64-73`** — two static field-definition arrays are re-allocated inline on every render of a form that re-renders per keystroke. They are pure data and belong in `consts`. Related nit at line 39: `mt={index && "0.5rem"}` passes the number `0` for the first item; `index === 0 ? 0 : "0.5rem"` is clearer.

**Correctness nits**

- **`.../Descriptions/index.tsx:26`** — `hasSelection` counts admin-class roles that line 60 then filters out, so a member whose only role is `ServiceDelivery` gets the "is going to receive…" header followed by nothing, and the "Select a classification…" hint is suppressed. (Distinct from Issue 26, which reaches the same empty-pane symptom via the locked-classification path.)
- **`.../Form/consts.tsx:51-53, 69`** — `this.parent as { divisionalHierarchyId?: string }` is an unchecked cast on a Yup `any`; if the value ever becomes a number, `nonAdminHierarchyIds.includes(...)` returns `false` and every classification is treated as admin — a validation bypass that fails open. The `?? []` at line 69 is dead (`buildRolesForSubmit` already defaults).
- **`api/divisionalHierarchy/getDivisionalHierarchy.ts:21-26`** — no caching, unlike the analogous `getMasterReferences.ts:29-35` which wraps the call in `getCachedValueWithFallback`. The response is requester-scoped, so any cache key must include `requesterId`; adding `staleTime` to `useDivisionalHierarchyQuery` is the cheaper option.
- **`GeneralOrderInfo/index.tsx:146`** — `groupDescription ?? undefined` can never fire (assigned a JSX literal at line 121).

**Dead code**

- **`.../FormikCheckboxesGroupedRoles/index.tsx:13, 17, 28`** — the `disabledRoles` prop is declared and wired but no caller passes it. Notable here because `isSelfEdit` disables the classification radio while leaving role checkboxes fully editable — either wire it up for self-edit or delete it.
- **`api/mixtures/users/search/forAssignment/const.ts:23-28`** — `ADMIN_ROLES` is now dead (its only import was removed; zero repo references). It documents the deprecated model the migration removes, so leaving it invites accidental reuse.
- **`api/users/search/forAssignment/searchForAssignment.ts:49, 59`** — `let orderAdminClassifications = new Set<string>()` is allocated on the non-admin path and never read there; the mixtures sibling scopes it correctly with `const`. Line 59's `as FetchUsersSearchByType` is a redundant cast that suppresses checking on the field controlling which population is fetched.

**Assessed clean:** new component folders otherwise use `index.tsx` / `hooks.ts` / `types.ts` correctly; shared atoms (`LoadingWrapper`, `RadioList`, `Separator`) are reused rather than reimplemented; `styles.ts` co-location is respected and `TeamMemberModal/styles.ts` correctly composes the shared `ModalWithConfirmation` styles rather than re-implementing them; the deleted `PARTNER_ACCESS_ROLES` has zero remaining consumers; `resolveAdminFlags.ts` has correct interface ordering, no casts and `!!` guards for null/undefined; `GeneralOrderInfo/utils.ts` handles the double-wrapped error shape defensively; `RolesFilter/utils.ts` has a sound dedupe and a documented `-1` tier sentinel; `settings/general/index.tsx` is a guard swap only, consistent with all six sibling partner pages; the SSR guard redirect shape is correct and no page in scope is left ungated or double-gated. No `as any` and no non-null assertions in the reviewed set. (Correction: the PR **does** add three `as unknown as` casts — in `guards.test.ts`, `RolesFilter/utils.test.ts` and `TeamMemberModal/__tests__/hooks.test.ts` — all in test files, all for context/fixture shaping.)

---

### 15. Role-overlap checks: evaluate `doArraysIntersect` vs `lodash/intersection` and settle on one

**Severity:** low

**Problem:** The role-overlap checks this PR introduces and touches (`onboarding/utils.ts:23`, `SettingsTilesList/utils.ts:14`, `MemberActions/utils.tsx:30`, `withUserProvided/utils.ts:44`) all use the repo's own `doArraysIntersect`. Lodash would also serve, and is available today with no new install — a direct dependency of both `apps/creative-portal` (`package.json:47`) and `packages/shared` (`package.json:52`), already imported at ~180 call sites. Worth deciding deliberately rather than by habit, so the codebase converges on one answer.

**Compare on these axes before choosing:**

| Axis               | `doArraysIntersect`                                                     | `lodash/intersection`                                                                          |
| ------------------ | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Return type        | `boolean` — what every call site here wants                             | `T[]` — needs `.length > 0` appended at each use                                               |
| Current footprint  | 14 invocations across 11 files, incl. `withAuthNeeded.ts:36`            | 193 lodash imports — and `intersection` is **already used**, at `TableWithFilters/hooks.ts:15` |
| Test coverage      | Own suite, 10 cases (`packages/shared/utils/doArraysIntersect.test.ts`) | Upstream                                                                                       |
| Equality semantics | `Array.includes`                                                        | SameValueZero, plus dedupes the result                                                         |
| Cost to switch     | —                                                                       | Repo-wide refactor touching auth middleware                                                    |

**Assessment:** the footprint row cuts against an assumption worth surfacing — `import { intersection } from "lodash"` already exists in this repo (`TableWithFilters/hooks.ts:15`), so adopting it would not be a new precedent. Even so, the evidence leans toward keeping `doArraysIntersect` — the boolean return matches every call site, it is already load-bearing in `withAuthNeeded.ts:36`, and it carries its own tests. But that is a judgement for the team, not something this review should decide unilaterally; the counter-case is that one fewer bespoke helper is one fewer thing to maintain. Either way, make it an explicit call and apply it consistently.

Note the semantic difference if the switch is made: `intersection` uses SameValueZero and dedupes, `doArraysIntersect` uses `Array.includes`. For the `UserRole` string enums used here they behave identically, so this is not a correctness concern in this codebase — but it would matter for `NaN` or duplicate-bearing inputs.

**Two dependency-hygiene corrections while evaluating:**

- **`@types/lodash` is _not_ a declared dependency.** Only `@types/lodash.throttle` is (`apps/creative-portal/package.json:92`). `@types/lodash` is present in `node_modules` transitively, and lodash ships no bundled types of its own. So the typings currently relied on by all ~180 existing lodash imports are undeclared — worth adding `@types/lodash` explicitly regardless of how this question is settled, since a dependency bump could remove it and break typecheck repo-wide.
- **`ramda` sits in `node_modules` but appears in no `package.json`** — transitive only. Don't import it directly here or in follow-ups.

---

### 16. Naming: several new identifiers contradict their own behaviour

**Severity:** low overall — but #1 below is a latent correctness bug, not a readability nit

**Problem:** Most of the naming in this PR is good. The cases below are ones where the name states something the code does not do, which is the category worth fixing — a merely awkward name costs a reader a second, a _wrong_ name sends them to the wrong conclusion. Reference counts are from the current branch, to size the effort.

**Rename recommended**

| #   | Current                                                                                       | Suggested                                                     | Problem                                                                                                                                                                                 |
| --- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- | ---------- | --- | ---------- | ----- | ----- | ------------------------------ | --- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `NON_ADMIN_CLASSIFICATION = "Creative"` (`api/divisionalHierarchy/consts.ts:5`, 7 refs)       | `CREATIVE_CLASSIFICATION`                                     | The name claims a _category_; the value is one _instance_. PP-1497's own hierarchy table already contains a second non-admin tier — verbatim `                                          | 6   | Accounting | 101 | Accountant | false | false | `, alongside `Service Delivery | 101 | Creative | false`. The day that is seeded, every `!== NON_ADMIN_CLASSIFICATION` check silently treats Accountants as admins. **Highest priority — this name becomes wrong, not merely unclear.** |
| 2   | `HIDDEN_DIVISIONS` / `HIDDEN_CLASSIFICATIONS` (`TeamMemberModal/utils.ts:10-11`, 2 refs each) | `NON_ASSIGNABLE_DIVISIONS` / `NON_ASSIGNABLE_CLASSIFICATIONS` | The comment directly above them says a member already holding one **is shown it, disabled** — so they are not hidden. Also aligns with the sole consumer, `isAssignableClassification`. |
| 3   | `forAdmin` (11 refs)                                                                          | `appliesToAdmin`                                              | An additive flag that reads as restrictive; currently needs the same clarifying comment twice.                                                                                          |
| 4   | `ADMIN_STEP_VISIBILITY` (`onboarding/const.tsx:12`, 7 refs)                                   | `COMMON_STEP_VISIBILITY`                                      | It is the shared creative **+** admin set, not admin-only. The current name makes `PERSONAL_INFORMATION` scan as admin-gated — directly relevant to the confusion in Issue 6.           |
| 5   | `RoleStepRelation` (8 refs)                                                                   | `OnboardingStepRule`                                          | No longer purely role-based once `forAdmin` exists.                                                                                                                                     |
| 6   | `ROLES_PER_ONBOARDING_STEP` (9 refs)                                                          | `ONBOARDING_STEP_RULES`                                       | Same staleness as #5.                                                                                                                                                                   |
| 7   | `CLIENT_API = "Client API"` (`GeneralOrderInfo/consts.ts:1`, 5 refs)                          | `CLIENT_API_CREATOR`                                          | It is a sentinel compared against `creator` (`hooks.ts:19`), but the name says nothing about creators.                                                                                  |
| 8   | `defaultEmptyNewMemberFormData` (4 refs)                                                      | `emptyNewMemberFormData`                                      | "default" and "empty" are redundant.                                                                                                                                                    |

**Low priority — judgement calls, fine to leave**

| Current                                                       | Suggested                    | Note                                                                                                                                                                                                |
| ------------------------------------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `staticRoleFilterOptions` (`RolesFilter/consts.ts:6`, 5 refs) | `STATIC_ROLE_FILTER_OPTIONS` | The only camelCase const among SCREAMING_SNAKE siblings (`OTHER_ROLES`, `CREATIVE_ROLES`, `ADMIN_CLASS_ROLES`). The repo is mixed overall, so this is consistency-within-feature, not a rule break. |
| `isItemVisible` (3 refs)                                      | `isItemVisibleFor`           | It is curried — `isItemVisible(user)` returns the predicate, it isn't one. But it reads fine at `.filter(isItemVisible(user))`, so debatable.                                                       |
| `RoleFilterOption` (with `isClassification`, 10 refs)         | —                            | Holds classifications _and_ roles under a "Role" name, but the UI filter is labelled "Roles", so it matches what users see. Leave it.                                                               |

**If only two are done: #1 and #2.** Both are cases where the name contradicts the behaviour rather than merely reading awkwardly, and #1 has a concrete future failure mode.

**Separately — the PR description is stale.** It references `deriveAdminRoles`, which does not exist in the branch (0 references). Role derivation was correctly dropped after Orlin's 2026-06-24 comment, but the description still describes the old design, which will mislead reviewers. Worth cleaning up before review.

These are mechanical renames; run typecheck and tests after.

---

### 17. `flagGuard` — correct, but four design gaps (one already required by the ticket)

**[File: apps/creative-portal/api/enhancers/withUserProvided/guards.ts]**

**Function/Class:** `buildGuard` / `flagGuard` / `administrativeGuard`

**Severity:** medium

**What is right:** the guard is fail-closed (no session user → redirect, covered at `guards.test.ts:41-45`); it runs _after_ every session/MFA/maintenance branch, as invoked at `withUserProvided/index.ts:169-171`, so the comment's ordering claim holds; the redirect shape is correct (`permanent: false`); and `handleCustomHandler` (`utils.ts:24-35`) correctly propagates the redirect and merges `user` into props on the pass path. `buildGuard` is the right factoring — both guards share one implementation, with 7 tests.

**Gap 1 — multi-flag gates cannot be expressed, and are already required.** `flagGuard` takes exactly one `AdminFlag`. But the 2026-06-30 decision comment says "Reporting: gate on `organizationAdmin` for now; **extend to `orderAdmin` later** if users report missing access", and Req 4.2.3 leaves main-layout/header as "flag(s) to be confirmed". The known-next requirement needs a `guards.ts` edit. A variadic signature keeps all 12 existing `flagGuard` call sites source-compatible (the 13th guard usage is `administrativeGuard`, which is unaffected):

```ts
export const flagGuard = (...flags: AdminFlag[]) =>
  buildGuard((user) => {
    const resolved = resolveAdminFlags(user);
    return flags.some((flag) => resolved[flag]);
  });
```

**Gap 2 — typed as `GetServerSideProps`, but crashes if actually used as one.** `flagGuard("userAdmin")` is directly assignable to a page's `getServerSideProps`, and TypeScript will not object: `apps/creative-portal/@types/session.ts:39-43` declares `session` as **non-optional** on `IncomingMessage`. It is only populated by `withUserProvided` at `index.ts:42`. Used standalone, `context.req.session.user` throws a TypeError → 500 instead of redirecting. The type system hides the footgun, and the export name reads like a usable gSSP. One-character fix, which also keeps it fail-closed:

```ts
hasAccess(context.req.session?.user);
```

**Gap 3 — not composable with page data.** `withUserProvided` has a single handler slot and `buildGuard` returns `{ props: {} }`, so a page cannot have both a flag gate and SSR data. That pattern already exists in this codebase (`onboarding/step1/index.tsx:106` fetches `stripePublishable`; `home/index.tsx:17` does redirect logic). All 13 current usages are pure gates, so nothing is broken today — but the partner-settings pages are exactly the kind that grow data needs:

```ts
const buildGuard =
  (hasAccess, handler?: GetServerSideProps): GetServerSideProps =>
  async (context) =>
    hasAccess(context.req.session?.user)
      ? handler?.(context) ?? { props: {} }
      : redirectToNoAccess;
```

**Gap 4 — API asymmetry.** `flagGuard("x")` is a factory call; `administrativeGuard` is a bare value. TypeScript catches a mistyped `administrativeGuard()`, but the inconsistency is free to remove.

**Test gap:** every case hand-builds `{ req: { session: { user } } }` and calls the guard in isolation. Nothing exercises it _through_ `withUserProvided`, so neither the ordering claim nor the props-merge is verified — precisely the coupling Gap 2 depends on. Add one integration test that runs a real page's `getServerSideProps` with a mocked session.

---

### 18. `createUser` is never awaited — the `try/catch` is dead code

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/hooks.ts]**

**Function/Class:** `onNewTeamMember`, lines 283–356

**Severity:** high

**Problem:** `createUser` is `mutateAsync` (line 54), which **rejects** on failure. It is called without `await`:

```ts
      try {
        createUser(createPayload, {
          onSuccess: async (createdUser: CreateUserResponse) => {
```

so the surrounding `catch (errors) { showDefaultErrorToast(errors); }` can never fire. The sibling `onEditTeamMember` awaits every mutation (lines 140, 181, 206), so the two submit handlers are inconsistent.

**Impact:** The rejection escapes as an unhandled promise rejection (dev overlay noise, Sentry noise). `onNewTeamMember` also resolves before the create completes, so Formik clears `isSubmitting` while the request is still in flight — the user can double-submit.

**Fix:** `await createUser(...)`, and drop the now-redundant `showDefaultErrorToast` in the catch since the `onError` callback already toasts:

```ts
      try {
        await createUser(createPayload, { onSuccess: ..., onError: ... });
      } catch {
        // Handled by the onError callback above
      }
```

---

### 19. The stale `classification` and admin flags are PUT back alongside the new `divisionalHierarchyId`

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/hooks.ts]**

**Function/Class:** `onEditTeamMember` (handler spans 105–262; payload built at 105–131), submitted via `updateUserProfile`

**Severity:** high

**Problem:** `updateUser` PUTs the whole object. `editedUser` is a `User`, which carries `classification?: string`, `orderAdmin?: boolean`, `organizationAdmin?: boolean` and `userAdmin?: boolean` (`api/users/types.ts:20, 33-34, 45`). None of them is destructured out in the handler's parameter list (lines 107–114), so `...editedUser` carries all three into the payload:

```ts
const updatedUser = {
  ...editedUser,
  ...values,
  roles: buildRolesForSubmit(roles),
  ...(numericDivisionalHierarchyId
    ? { divisionalHierarchyId: numericDivisionalHierarchyId }
    : {})
};
```

**Impact:** There is no field-picking anywhere on the path — `putUser.ts:45` does `const userPutData: User = { ...body }` and `api/utils/users/updateUser.ts:9` forwards the whole object to the member service. So demoting a member from Managing Editor to Creative sends `divisionalHierarchyId: <creative id>` **and** `classification: "Managing Editor"` **and** `orderAdmin: true` in the same request. Which wins is entirely up to the backend — precisely the ambiguity the comment at lines 117–120 claims to avoid. It also silently reverts a classification change made elsewhere between the lookup and the save.

**Fix:** Strip the server-derived fields the same way roles are stripped, ideally via a shared, testable constant in `utils.ts`:

```ts
const {
  classification: _classification,
  orderAdmin: _orderAdmin,
  organizationAdmin: _organizationAdmin,
  ...editedUserWithoutDerivedAccess
} = editedUser ?? {};
```

---

### 20. `searchForAssignment` — empty hierarchy silently returns zero admins, peers are excluded, and it now fetches every active user

**[File: apps/creative-portal/api/users/search/forAssignment/searchForAssignment.ts]**

**Function/Class:** handler, lines 49–88 (and the sibling `api/mixtures/users/search/forAssignment/searchForAssignment.ts:58-78`)

**Severity:** high

**Problem:** Three claims about the migrated assignment path. The handler itself has no test; the extracted helpers do (`api/utils/users/orderAdminAssignment.test.ts`, 5 tests, including an empty-hierarchy case).

(a) `getOrderAdminClassifications` builds a `Set` from `hierarchy.filter((entry) => entry.orderAdmin)`. If `fetchDivisionalHierarchy` returns `[]` — requester lacks hierarchy visibility, BE returns empty, or a shape change drops `orderAdmin` — the Set is empty, `isOrderAdminClassification` returns `false` for every user, and the endpoint returns `200` with an empty array. No log, and indistinguishable from "genuinely no admins".

(b) **⚠️ Unverified — treat as a question for the BE, not a defect.** This rests entirely on comments the PR itself wrote (`RolesFilter/utils.ts:7-14`, `TeamMembersTable/hooks.ts:49-52`) stating that §4.1 returns classifications **beneath** the requester and omits their own. Nothing in the repo corroborates it. So an order admin at the _same_ classification as the requester can never appear as an assignment candidate. The previous `ADMIN_ROLES` role search had no tier restriction — this is a silent narrowing that drops valid assignees.

(c) The previous implementation issued three narrow `searchBy: "roles"` queries. It now pulls the entire active-user table — **on the `isAdminRole` branch only** (lines 51–67); every other role still issues one narrow `searchBy: "roles"` query at 68–74 — and discards most of it client-side:

```ts
fetchUsers({
  requesterId: requesterId.toString(),
  searchBy: "status" as FetchUsersSearchByType,
  userStatus: "A"
}),
```

`fetchUsers` has no pagination or limit, so for admin-role requests size and latency now scale with total headcount rather than admin headcount.

**Impact:** Assignment dropdowns render empty or silently miss peers, with no error surfaced; and the endpoint's cost grows with company size.

**Fix:** Log a warning when `orderAdminClassifications.size === 0`; confirm with the BE whether §4.1 should be unioned with the requester's own entry (this determines whether (b) is a bug or intended); and push the filter server-side rather than fetching all active users. Add tests for the handler itself — given (a) and (c) live there, it remains the highest-leverage test addition in the PR.

---

### 21. The "All Roles" label is now unreachable — silent UI regression

**[File: apps/creative-portal/components/molecules/tables/TeamMembersTable/consts.tsx]**

**Function/Class:** roles `Cell`, lines 152–162

**Severity:** medium

**Problem:** `roleItems` is now filtered through `ADMIN_CLASS_ROLES` (`{ServiceDelivery, ServiceSupport, Admin, Superadmin}`), but the comparison target was left alone:

```ts
      if (
        roleItems.length === extendedTaskRolesWithoutAdminTypes.length
      ) {
```

`extendedTaskRolesWithoutAdminTypes` is `[Editor, Returner, Reviewer, ServiceDelivery, ServiceSupport]` — length **5**, two of which `roleItems` can never contain. `UserRole` has 8 members and `ADMIN_CLASS_ROLES` removes 4, so `roleItems` holds at most `{Editor, Reviewer, Returner, QA}` — a maximum of **4**, never 5. The branch can never fire. Before this PR the comparison ran against the unfiltered array and did match.

**Impact:** A member holding every task role renders a 3-item `ExpandableList` instead of the compact "All Roles" / "Managing Editor, All Roles" label. No test covers it.

**Fix:** Compare against the roles that can actually survive the filter, and export the derived list so the two can't drift again:

```ts
export const nonAdminTaskRoles =
  extendedTaskRolesWithoutAdminTypes.filter(
    (role) => !ADMIN_CLASS_ROLES.has(role)
  );
```

---

### 22. `canManageMember` fails open, and the disabled query makes that the default for non-user-admins

**[File: apps/creative-portal/components/molecules/tables/TeamMembersTable/utils.ts]**

**Function/Class:** `canManageMember`, lines 13–19

**Severity:** medium

**Problem:**

```ts
export const canManageMember = (
  classification: string | undefined,
  manageableClassifications: Set<string>
) =>
  manageableClassifications.size === 0 ||
  !classification ||
  manageableClassifications.has(classification);
```

Two open doors, both documented as deliberate in the comment above. (a) `size === 0` → everyone manageable; but `hooks.ts:37-42` sets `enabled: userAdmin`, so for any non-user-admin the set is _permanently_ empty and `isInitialLoading` is false — no skeleton, row actions shown on every member. (b) `!classification` → a member with no classification is manageable by anyone regardless of tier, which is exactly the case the migration exists to gate.

**Impact:** Not full privilege escalation — destructive items are still gated inside `MemberActions` by `useAdminFlags().userAdmin` — but it contradicts the stated "you can only manage people beneath you" rule and leaves a live menu on peer rows.

**Fix:** Distinguish "unresolved" from "permitted" rather than conflating them:

```ts
export const canManageMember = (
  classification: string | undefined,
  manageableClassifications: Set<string>,
  isHierarchyResolved: boolean
) =>
  !isHierarchyResolved ||
  (!!classification && manageableClassifications.has(classification));
```

---

### 23. `filteredUsers` is computed, logged, then thrown away

**[File: apps/creative-portal/api/mixtures/users/search/forAssignment/searchForAssignment.ts]**

**Function/Class:** handler, lines 159–173

**Severity:** medium

**Problem:** The comment says "Filter users to only include those who belong to at least one of the requested organization groups", the log reports `filteredUsers.length` — and the **unfiltered** array is returned:

```ts
    const filteredUsers = usersWithAssignedJobs.filter(
      (user) => user.organizationGroupId !== undefined
    );

    logger.info(..., { userCount: filteredUsers.length });

    return res.status(200).json(usersWithAssignedJobs);
```

**Impact:** Clients receive more users than intended, and the log understates the response, so the discrepancy is invisible in observability. Pre-existing (outside this PR's hunks), but it now sits in the migrated admin path where the candidate pool just grew to "all active users" per Issue 20(c) — so the over-return is materially larger than before.

**Fix:** `return res.status(200).json(filteredUsers);` — or delete `filteredUsers` and log the real count if returning everything is intended.

---

### 24. Additional medium-severity code-quality findings

**Severity:** medium

- **`TeamMemberModal/hooks.ts:206` — `updatedUser as User` masks a real `undefined` path.** `editedUser` is `User | undefined`; every other use in the handler guards it (lines 135, 177–180), this one asserts instead. If the lookup hasn't resolved or 401s, `updatedUser` has no `id` and `updateUser` calls `apiRoutes.userById(undefined)`. Add `if (!editedUser) return;` at the top of the handler — the cast and both redundant guards then disappear.
- **The Formik context is typed four incompatible ways.** `hooks.ts:37-40` (`TeamMemberFormValues`), `Form/index.tsx:17-20` (`{ roles?: string | string[]; phone?: string }`), `ClassificationRadioGroup/hooks.ts:27` (`{ classification?: string }`), `Descriptions/index.tsx:15` (`TeamMemberProps`). None matches the runtime bag, and `TeamMemberProps` has no `classification` field even though `ClassificationRadioGroup/hooks.ts` depends on it (`Descriptions` reads `divisionalHierarchyId`, which _is_ on the type). Declare the value shape once in `Form/types.ts` and consume it everywhere.
- **`ADMIN_CLASS_ROLES` filtering is duplicated at four sites** instead of using `buildRolesForSubmit`: `Descriptions/index.tsx:60`, `settings/personal-info-content/utils.ts:19`, `TeamMembersTable/consts.tsx:137`, plus the util itself. Worse, a page and a table now `import { ADMIN_CLASS_ROLES } from "components/organisms/modals/TeamMemberModal/utils"` — reaching into an organism's private utils. Per CLAUDE.md's reuse-first rule this belongs in `packages/shared` (or at least `apps/creative-portal/utils/`).
- **`Form/index.tsx:43-63` infers edit-vs-create mode from `hasOwnProperty` probes on `initialValues`.** The parent already knows (`editedUserId`). The `adminLegacyId`/`editorLegacyId` probe is inert — `defaultEmptyNewMemberFormData` includes both keys, so it is always true. Pass `isEditMode` explicitly alongside the existing `isSelfEdit`.
- **`TeamMembersTable/consts.tsx:203-204` calls `useAdminFlags()` per row inside a react-table `Cell`,** while `hooks.ts:32` already holds `userAdmin` and threads the sibling values down as params. Rules-of-hooks is satisfied only incidentally; enabling `react-hooks/rules-of-hooks` on this file would break the build. Pass it in like the others.
- **`RolesFilter/index.tsx:24-26` — `option as RoleFilterOption` erases `null`.** react-select's `onChange` is `(newValue: Option | null)`. Destructuring throws the moment the select is cleared (backspace on empty input, programmatic clear, or a future `isClearable`). Guard for `null` before destructuring.
- **`GeneralOrderInfo/index.tsx:60-100` still holds `useRef` + `useMemo`** despite this PR creating a sibling `hooks.ts` and moving the query logic there — the "index.tsx is UI-only" refactor is half-applied. Move both into `useGeneralOrderInfo`.
- **`Number()` conversions have no `NaN` guard** (`hooks.ts:121-123, 280`). `Number("abc")` → `NaN`, which `JSON.stringify` serialises as `null`; the create path has no guard at all, since Yup's `.required()` checks presence, not numericness. The edit path additionally drops a legitimate id of `0` via the truthiness check. Use `Number.isFinite`.

---

### 25. "Non-admin" is defined two contradictory ways within the same feature

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/hooks.ts]**

**Function/Class:** `nonAdminHierarchyIds` (lines 95–103) vs `ClassificationRadioGroup/hooks.ts` `orderRank` (lines 12–14)

**Severity:** medium

**Problem:** Two files in the same feature classify hierarchy entries by incompatible rules.

```ts
// TeamMemberModal/hooks.ts:98 — flag-derived
.filter((entry) => !entry.orderAdmin && !entry.organizationAdmin)

// ClassificationRadioGroup/hooks.ts:12-14 — tier-derived
const END_USER_TIER_FLOOR = 100;
const orderRank = (tier: number) => (tier > END_USER_TIER_FLOOR ? -1 : tier);
```

The tier form matches what the code documents as the BE's definition — _"1 - 100: User Admin / 101 - ∞: End User"_. ⚠️ That quote is sourced from the PR's own comment at `ClassificationRadioGroup/hooks.ts:10` and could not be corroborated independently, so treat the tier boundary as needing BE confirmation. The flag form cannot, because §4.1 returns only `orderAdmin` and `organizationAdmin`; `userAdmin` is **not** in the response (confirmed in `DivisionalHierarchyEntry`, which correctly omits it).

**Impact:** Any classification inside the user-admin tier band with both order/org flags false is a genuine user-admin that `nonAdminHierarchyIds` treats as non-admin, forcing a role selection the BE does not require. Believed latent on current seed data — unverifiable from the repo, and the only in-repo Creative fixture (`ClassificationRadioGroup/hooks.test.ts:44`) sets `orderAdmin: true`, contradicting that assumption — so this is a correctness-of-reasoning defect rather than a live bug. It becomes live the moment a non-order/org admin tier is seeded, which is exactly what PP-1497's "Creative manager at tier 25" example describes.

**Fix:** Use the BE's rule at both sites and hoist the constant so there is one definition:

```ts
// api/divisionalHierarchy/consts.ts
export const END_USER_TIER_FLOOR = 100;
export const isEndUserTier = (entry: DivisionalHierarchyEntry) =>
  entry.tier > END_USER_TIER_FLOOR;
```

Then `nonAdminHierarchyIds` filters on `isEndUserTier` and `orderRank` reuses the same constant. Note the fixture caveat recorded under Tests: the `ClassificationRadioGroup/hooks.test.ts` fixtures use tiers `10`, `30` and `50` — none above `END_USER_TIER_FLOOR` — so neither rule is exercised at a tier where they would diverge.

---

### 26. The description panel contradicts the radio on the locked-classification path

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/partials/Descriptions/index.tsx]**

**Function/Class:** `TeamMemberDescriptions` (component from line 14; derivation at 18–26)

**Severity:** medium

**Problem:** `Descriptions` resolves the selected classification by id off the form values:

```ts
const selectedClassification = hierarchy.find(
  (entry) => String(entry.id) === values.divisionalHierarchyId
)?.classification;
```

But when `lockedClassification` is active, `ClassificationRadioGroup/index.tsx:34-44` renders a **non-Formik** `RadioList`, so `divisionalHierarchyId` stays `""`. The lookup misses, `classificationInfo` is undefined, and `hasSelection` is false unless the member holds any role at all — line 26 reads `values.roles` unfiltered, so even an admin-class role stripped at line 60 flips it true (see the related nit in Issue 14).

**Impact:** The left pane shows the member's classification rendered and checked; the right pane simultaneously reads _"Select a classification and roles to see their description."_ Two panels of the same modal disagree about whether a classification is selected. This is the same root cause as the untested "looks selected but isn't sent" branch already noted under Tests, surfacing as a visible UI contradiction rather than a silent payload gap.

**Fix:** Have `Descriptions` fall back to `lockedClassification` — cleanest once the duplicated derivation is lifted into the shared `useTeamMemberModal`/`utils` layer per Issue 24's Formik-typing point, so both partials read one source of truth instead of each re-deriving from `values`.

---

### 27. Blank Settings tab for admin-class members — pre-existing, not caused by this PR

**[File: apps/creative-portal/components/pages/team-members/profile/[memberId]/partials/SettingsTab/consts.tsx]**

**Function/Class:** `getSettingColumns` — 8 of 10 items gated `roles: ALL_ROLES` with no `showToAdmin`

**Severity:** medium (downgraded from high — see the correction below)

**Problem:** Opening the profile of a Service Delivery Manager, Managing Editor or Service Delivery Support renders a **blank Settings tab**: 8 of 10 tiles (Notes, General, Languages, Teams, Editor Profile among them) are gated `roles: ALL_ROLES` with no `showToAdmin` or `adminFlag`, and an admin-class member holds no functional roles, so `doArraysIntersect(ALL_ROLES, [])` is false for every one. Viewing a Creative still works (they hold `Editor`), so it survives casual testing.

> **⚠️ Correction — this PR does not cause it.** An earlier draft of this review claimed the migration introduced the regression. Verification refuted that. The base filter was `.filter(({ roles }) => doArraysIntersect(roles, userRoles))` with `userRoles = data?.roles || []` (`SettingsTilesList/index.tsx:26` at `cc44dd6`). The new `isItemVisible` opens with the behaviourally identical `doArraysIntersect(roles ?? [], user?.roles ?? [])` (`utils.ts:14`) and then adds **two further ways to pass** (`showToAdmin`, `adminFlag`). It is therefore _strictly more permissive_ than what it replaced — the PR cannot have made anything less visible. `SettingsTab/index.tsx:29` changing `roles={data?.roles || []}` → `user={data}` is a signature adaptation, not a semantic one. A member with empty `roles` saw a blank tab before this PR too. The earlier claim that `ALL_ROLES` "now means a strictly smaller set" was also wrong — it meant exactly that before.

**Impact:** A real, user-visible gap that becomes _more_ prominent as Rev 4.0 moves admins off functional roles — but a pre-existing defect this PR had the opportunity to close, not a regression it introduced. It does not block this merge.

**Fix:** Apply the correction the PR already makes in the sibling file — `settings/consts.tsx:114-115` gained `showToAdmin: true` — to all eight `ALL_ROLES` items in `SettingsTab/consts.tsx`. Reasonable either here (small, in-theme) or as a follow-up. A test covering an admin-class member (empty `roles`, `userAdmin: true`) would also close the untested-`isItemVisible` gap recorded under Tests.

---

## Validation Checks

| Check                     | Result                   | Notes                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `npx turbo run test`      | ⚠️ Pre-existing failures | Creative-portal: **33 test files fail to collect, 1502/1502 collected tests pass**. Customer-portal: 6 files fail. Root cause is `ERR_REQUIRE_ESM` — `@theme-ui/mdx` requiring `@mdx-js/react` in `node_modules`. **Not caused by this PR** (see below). 13 of the PR's own test files pass; the PR touches 18 test files in total (13 added, 5 modified). |
| `npx turbo run typecheck` | ✅                       | 5/5 workspaces, 0 errors                                                                                                                                                                                                                                                                                                                                   |
| `npx turbo run lint`      | ✅                       | 5/5 workspaces, 0 errors                                                                                                                                                                                                                                                                                                                                   |
| `npx turbo run build`     | ⚠️ Pre-existing failure  | `@proofed/customer-portal#build` fails with the same `ERR_REQUIRE_ESM` from `@theme-ui/mdx` → "Failed to collect page data for /orders". Shared + wysiwyg build clean.                                                                                                                                                                                     |

**Evidence the failures are pre-existing and unrelated:**

- The PR modifies **no** `package.json`, `yarn.lock`, `turbo.json`, `vitest` or `tsconfig` file — it cannot have caused a `node_modules` ESM/CJS resolution error.
- The PR touches no `customer-portal` file at all (consistent with Jira §6: "Customer Portal is unchanged").
- ⚠️ **Correction to an earlier draft of this section:** it claimed the PR touches _none_ of the failing files. That is wrong — three files in the failing set are PR-modified or PR-cited: `TeamMembersTable/utils.test.ts`, `OrderManagment/partials/GeneralOrderInfo.test.tsx`, and `MemberActions/utils.test.tsx` (cited as evidence in the Tests section below). The root-cause argument is unaffected — they fail at **collection** time with the same `ERR_REQUIRE_ESM`, before any test in them runs — but the supporting evidence as originally stated was not accurate.
- The failures are collection-time import errors (0 tests executed in those files), not assertion failures.

The 13 added test files all pass:
`TeamMemberModal/__tests__/hooks.test.ts` (14), `TeamMemberModal/utils.test.ts` (7), `Form/consts.test.ts` (7), `ClassificationRadioGroup/hooks.test.ts` (3), `guards.test.ts` (7), `orderAdminAssignment.test.ts` (5), `GeneralOrderInfo/utils.test.ts` (8), `personal-info-content/utils.test.ts` (5), `resolveAdminFlags.test.ts` (4), `useAdminFlags.test.ts` (3), `onboarding/utils.test.ts` (5), `isAdministrative.test.ts` (4), `RolesFilter/utils.test.ts` (6).

Per CLAUDE.md, these pre-existing failures should still be flagged to the team and resolved before anything merges on top of them.

---

## Tests

- ✅ Good unit coverage of the new pure logic: `resolveAdminFlags`, `useAdminFlags`, `isAdministrative`, `guards`, `RolesFilter/utils`, `onboarding/utils`, `TeamMemberModal/utils`
- ✅ AC (d) — SD/SS not submitted in `roles` — covered at `utils.test.ts:31-41` and `__tests__/hooks.test.ts:224`
- ✅ AC — `divisionalHierarchyId` submitted on **Update** — `__tests__/hooks.test.ts:246` asserts `toBe(1003)`
- ❌ AC (c) — `divisionalHierarchyId` submitted on **Create** — **not asserted anywhere**. `newMemberValues` gained `divisionalHierarchyId: "1006"` (line 373) but no test reads it back off the payload. The `Number(divisionalHierarchyId)` conversion at `hooks.ts:280` is entirely unpinned — if it regressed to `NaN` or was dropped, every test still passes. This is Jira testing-note #3 and #5, on a BE-required property.
- ❌ AC (b) — classification required on both Create and Edit — not met, and `consts.test.ts:75-84` actively locks in the deviation (see Issue 2)
- ⚠️ AC (a) — System Administration excluded — unit-tested via `isAssignableClassification` (`utils.test.ts:75-81`), but `ClassificationRadioGroup/hooks.test.ts` never feeds a System Administration entry through the hook (every fixture uses `division = "Service Delivery"`). The end-to-end path is unverified for the one division the AC names.
- ⚠️ AC (e) — `userAdmin=false` cannot open the modal — SSR guards covered (`guards.test.ts:27-31`), but `MemberActions/utils.test.tsx` passes `hasRequiredRolesToEdit: true` in all four cases, so the `false` branch — the actual AC — is never exercised
- ❌ No tests for the entire API/service layer: `getDivisionalHierarchy.ts`, `fetchDivisionalHierarchy.ts`, `services/divisionalHierarchy/index.ts`. The service carries a load-bearing `retry: false` (because a plain requester gets a 401) that nothing pins
- ❌ No rendering test for `ClassificationRadioGroup/index.tsx`. Untested branches include the `lockedClassification` path (lines 34–44), which renders a non-Formik `RadioList` whose value is **never submitted** — exactly the "looks selected but isn't sent" behaviour that needs a test
- ❌ `SettingsTilesList/utils.ts` (`isItemVisible`) is new and untested despite three distinct visibility branches
- ❌ `ClassificationRadioGroup/hooks.ts:12-14` documents `END_USER_TIER_FLOOR = 100` so "None"/Creative sorts first. The fixtures are tier `10` (SD Head), `30` (Managing Editor) and `50` (Creative) — **none exceeds 100**, so the `-1` sentinel never fires. Worse, Creative therefore sorts **last** in all three assertions, which is the opposite of the documented design intent. The ordering rule has zero coverage and the fixture contradicts the design. (An earlier draft said "every fixture uses tier 50" — only one does.)
- ❌ No tests pin the _preserved_ behaviour on the Settings/Onboarding/Availability surfaces changed in Issues 6 and 7
- ❌ Manual testing, E2E, UI-vs-design check, and DevTools API verification all unchecked in the PR description

---

## Summary

| Aspect           | Status                                                                                                                                                                                                                                                                        |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Correctness      | ❌ Five high-severity defects (irreversible admin lockout, Edit validation gap, inconsistent flag read, unawaited create, stale fields in the update payload). Issue 20's assignment-search claims are partly unverified; Issue 27 was refuted on verification and downgraded |
| Regression risk  | ❌ High — access-control migration touching 127 files, with four surfaces changed against an explicit "no change" decision, plus one confirmed silent regression (Issue 21); Issue 20b is unverified and Issue 27 proved pre-existing                                         |
| Tests            | ⚠️ Good coverage of new pure logic; API/service layer, both `searchForAssignment` handlers, and two named ACs untested; one test locks in a requirement deviation                                                                                                             |
| Code quality     | ⚠️ Sound abstractions and faithful to existing patterns, but the file-by-file pass surfaced duplication, four incompatible Formik context types, and ~20 convention deviations                                                                                                |
| Validation suite | ⚠️ typecheck + lint clean; test + build fail on a **pre-existing, unrelated** `node_modules` ESM issue — all of the PR's own tests pass                                                                                                                                       |
| Mergeable state  | ✅ Clean (GitHub), but PR is a **draft** with manual testing, E2E, design check and `yarn bump-packages` unchecked                                                                                                                                                            |

---

## Recommendation

**Request changes.**

Must fix before merge:

0. **Issues 18–20 — three defects found on the final file-by-file pass.** `createUser` is never awaited, so its `try/catch` is dead and the create can double-submit (Issue 18). The edit payload PUTs the stale `classification` and admin flags alongside the new `divisionalHierarchyId`, leaving the backend to arbitrate (Issue 19). And `searchForAssignment` — which has **no tests at all** — silently returns zero admins when the hierarchy is empty, structurally excludes same-tier peers, and now fetches every active user per panel open (Issue 20).
1. **Issue 1 — remove `Admin`/`Superadmin` from `ADMIN_CLASS_ROLES` on the submit path.** This is the blocker: editing any System Administrator strips the only role that keeps them admin, and nothing in the UI can restore it. Update the test at `__tests__/hooks.test.ts:227-248` to assert preservation.
2. **Issue 2 — require `divisionalHierarchyId` on the Edit schema _only when `isSelfEdit` is false_,** and narrow the empty-value escape hatch to the genuinely non-grantable case, so a failed hierarchy fetch cannot submit a PUT missing a BE-required property. ⚠️ Do **not** add an unconditional `.required()` — PP-1934 (2026-07-03) made the field conditional and self-updaters must omit it, so an unconditional rule would reintroduce the self-update failure that ticket was raised to fix. Update `consts.test.ts:75-84` and add self-edit vs other-edit payload assertions.
3. **Issue 3 — route `MainLayout` through `useAdminFlags()`** instead of reading `user.organizationAdmin` directly.
4. **Resolve the requirement conflicts with Orlin (Issues 6 and 7)** — Settings, Onboarding and Availability were fenced off by the 2026-06-30 decision comment but changed here. Either revert to preserve behaviour or get the decision amended, and add tests either way. Note the onboarding half is a rollout-window risk rather than a flat regression (see the correction in Issue 6), and the proposed remedy belongs in `resolveAdminFlags`, not in the onboarding roles list.
5. **Confirm the §5.1 flag property names against the deployed service** before merge — the PR description itself lists this as unverified (`organizationAdmin` vs `orgAdmin`), and Issue 4 shows a mismatch fails silently and closed.
6. **Add the missing Create-path assertion** that `divisionalHierarchyId` reaches the payload as a number.

Should fix:

8. Issue 5 (nav advertises inaccessible pages) — the most visible day-one symptom for Returners and single-flag admins.
9. Issue 8 (string-based System Admin filter fails open), Issue 9 (`enableReinitialize` data loss), Issues 10–12 (Created-by fallback robustness).
10. Issue 13 — split the Order-sidebar work into its own PR; Jira §6 puts it out of scope.
11. Issue 27 — add `showToAdmin: true` to the eight `ALL_ROLES` items in `SettingsTab/consts.tsx`. Pre-existing rather than PR-caused (see the correction in that issue), so optional here.
12. Issue 25 — settle on the BE's tier-based definition of "non-admin" and hoist `END_USER_TIER_FLOOR` so the modal hook and the radio group stop disagreeing.
13. Issue 26 — make `Descriptions` fall back to `lockedClassification` so the two modal panes stop contradicting each other.
14. Issue 14 conventions, and the missing API/service and `ClassificationRadioGroup` render tests.
15. Issue 16 renames #1 (`NON_ADMIN_CLASSIFICATION` → `CREATIVE_CLASSIFICATION`) and #2 (`HIDDEN_*` → `NON_ASSIGNABLE_*`) — #1 in particular becomes an actual bug the moment a second non-admin classification is seeded.
16. Issue 15 — make an explicit team call on `doArraysIntersect` vs `lodash/intersection`, and declare `@types/lodash` explicitly regardless of the outcome.
17. **Refresh the PR title and description.** The title still says "removable stub" while the body states there is no stub; the body still describes the dropped `deriveAdminRoles` design; and the open-questions list should be updated to record that PP-1934 resolved the Lookup `classification` and conditional-`divisionalHierarchyId` questions.

Also note: the pre-existing `@theme-ui/mdx` / `@mdx-js/react` `ERR_REQUIRE_ESM` failure breaks 33 creative-portal test files and the customer-portal build on `develop` as well. It is not this PR's doing, but per CLAUDE.md it should be raised with the team and fixed before merging on top of it.
