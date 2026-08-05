# PR Review: chore/PP-1890: remove Returner role from team-member UI; route return jobs to admins

**PR:** https://github.com/Proofed/B2BWebserver/pull/2396
**Jira:** https://proofed.atlassian.net/browse/PP-1890
**Status:** Jira not fetched — access was not authorized this session. Requirements below are **inferred** from the PR description, commit messages, and code, not read from the ticket.

---

## Jira Requirements vs Implementation

> Inferred requirements (PP-1890 / OMS Rev 4.0 role model). Verify against the actual ticket before merge.

| Requirement (inferred) | PR Implementation | Status |
|---|---|---|
| Remove the Returner role from user-facing surfaces (add/remove/filter) | Removed from `TeamMemberModal` Form ("Other Roles" group + checkbox), `NewTeamMemberModal` options, `RolesFilter` static options, `TaskRoleDropdown` sets, `createUserGroupMembers` schema/types | ✅ Addressed |
| Preserve existing Returner holders (hide-only) | `ADMIN_CLASS_ROLES` = `{Admin}` only; `buildRolesForSubmit` keeps any non-Admin role (incl. a legacy `"Returner"` string), so it is never stripped on edit | ✅ Addressed |
| Route return jobs to order-admins | `getRoleNameBasedOnJob(RETURN)` → `UserRole.Admin`; single-search `isAdminRole` branch resolves order-admins by classification `orderAdmin` flag | ✅ Addressed |
| Return jobs carry no assessments | Bulk path skips per-user assessment fetch when `jobType === RETURN`; single path already returns none for `Admin` searches | ✅ Addressed |
| Admin access driven by DivisionalHierarchy flags, not roles | `resolveAdminFlags` now flag-only; `isAdministrative` = flags only; `useAdminFlags`/guards updated | ✅ Addressed |
| Identify System Admin without the retired Superadmin role | `isSystemAdmin` now keys off `classification === "System Admin"` | ⚠️ Addressed but depends on an unverified backend contract (see Issue 1) |
| Retire deprecated roles from the type system | `Superadmin`, `ServiceDelivery`, `ServiceSupport`, `Returner` removed from `UserRole` enum | ✅ Addressed |
| Team-table ordering under the new model | New `makeTeamMemberComparator` sorts by status → classification tier → creative role → name | ✅ Addressed |

**Scope note:** The PR body frames this as "remove Returner from the team-member UI," but the diff is substantially larger — it removes **four** roles from the `UserRole` enum and re-architects admin gating from role-based to flag/classification-based (access control). See Issue 3.

---

## Architecture Analysis

The change completes the OMS Rev 4.0 migration from role-membership gating to DivisionalHierarchy flags:

- **Source of truth flip.** `resolveAdminFlags` previously OR'd the BE flags with a legacy `Admin`/`Superadmin` → all-flags fallback. It now returns the raw BE flags only. `isAdministrative` drops its `Returner` special-case. `isSystemAdmin` moves from "holds the Superadmin role" to "classification is System Admin."
- **Return-job routing.** `getRoleNameBasedOnJob` maps `RETURN → Admin` and its second argument changed from `roles?: UserRole[]` to `isOrderAdmin = false`; both live callers (`ChangeJobStatusModal`, `BulkAssignmentModal`) now pass `orderAdmin` from `useAdminFlags()`. The third caller (`pages/jobs/index.tsx`) passes no second arg and only uses the result for a conflict-toast message — behavior preserved.
- **Assessment skip.** Threaded `jobType` (enum-validated, optional) into the bulk search only. This is correct: the single-search Admin branch filters group members by `taskRoles.includes("Admin")` (always empty → no assessments), whereas the bulk path builds `currentUserGroupMemberIds` from *all* members, so it genuinely needs the explicit `RETURN` skip to avoid fetching QA assessments for return jobs.
- **Enum cleanup.** All four removed enum members have zero remaining source references (grep across `apps` + `packages`, excluding tests/mocks) — typecheck passes, confirming no dangling `UserRole.X`.

The approach is coherent and the callers are consistently updated. The principal risk is not in the code shape but in the backend contract the new gating relies on (Issues 1 & 2).

---

## Issues Found

### 1. System-admin detection depends on a backend contract the removed code explicitly said did not hold

**[File: apps/creative-portal/utils/resolveAdminFlags.ts]**

**Function/Class:** isSystemAdmin

**Severity:** medium

**Problem:** `isSystemAdmin` now returns `user?.classification === SYSTEM_ADMIN_CLASSIFICATION` (`"System Admin"`). The code this PR deletes stated the opposite as fact: *"the Authentication response (§5.1) carries no `classification` at all … the System Administration classification is redacted (never returned by the API) … so classification cannot identify a system admin"* — which is exactly why the old implementation kept the `Superadmin`-role signal. Nothing in the diff demonstrates that the auth response now returns `classification` for a system admin, or that the value is the exact string `"System Admin"`. Note also the naming split in the codebase: the *division* is `"System Administration"` (`NON_ASSIGNABLE_DIVISIONS`), while this const is `"System Admin"` — an easy mismatch if the BE returns the division name or different casing.

**Impact:** If the auth response omits `classification` for system admins (as the old comment asserted) or uses a different string, `isSystemAdmin` returns `false` for everyone. The system-admin-only surfaces gated on it — notably Refresh Cache (`api/cacheManagement/removeAllCache.ts`) and the Header control — become inaccessible to all users. No unit test can catch this; it depends entirely on the live OMS backend. The PR's "Manual testing completed" box is unchecked.

**Fix:** Verify against the OMS Rev 4.0 auth response in devtest that a system admin's session carries `classification` with the exact value in `SYSTEM_ADMIN_CLASSIFICATION`, and confirm the string matches what the BE returns (division vs classification, casing). Capture the evidence in the PR before merge. If the auth payload does not carry it, this gate needs a different signal.

### 2. `resolveAdminFlags` removes the safety-net fallback, so all admin access now hinges on BE-supplied flags

**[File: apps/creative-portal/utils/resolveAdminFlags.ts]**

**Function/Class:** resolveAdminFlags

**Severity:** medium

**Problem:** The function previously OR'd the BE flags with a legacy `Admin`/`Superadmin` → all-flags fallback, deliberately described as a *"safety net, ensuring a system admin is never locked out."* It now returns `{ orderAdmin, organizationAdmin, userAdmin }` straight from the user object. The new comment asserts *"a System Admin carries all three, so no role or classification fallback is needed"* — an assumption about backend behavior, not something enforced or verified in this repo.

**Impact:** Same class of risk as Issue 1. If the BE does not populate all three flags for system admins (or for users previously covered by the `Admin`/`Superadmin` fallback), those users silently lose access to team-management (`userAdmin`), organization-management (`organizationAdmin`), and order surfaces gated on the flags. Combined with Issue 1, both the "everyone" system-admin gate and the per-capability gates lose their previous fallback simultaneously.

**Fix:** As part of the same devtest verification, confirm the auth response sets `userAdmin`/`orderAdmin`/`organizationAdmin` for the System Administration classification (and any other classification expected to have admin access). This and Issue 1 are the two things to prove before merge.

### 3. PR description understates the scope (access-control changes are not surfaced)

**[File: (PR description)]**

**Function/Class:** —

**Severity:** low

**Problem:** The PR title/body describe removing the Returner role and routing return jobs to admins. The diff additionally removes `Superadmin`, `ServiceDelivery`, and `ServiceSupport` from the `UserRole` enum and changes `resolveAdminFlags`/`isSystemAdmin`/`isAdministrative` from role-based to flag/classification-based gating. The commit history makes this clear, but a reviewer reading only the PR body would not know the blast radius touches system-admin access control.

**Impact:** Reviewer/QA may under-test the highest-risk area. Reduced auditability of an access-control change.

**Fix:** Expand the PR description to call out the enum retirement of all four roles and the flag/classification-based admin-gating migration, and explicitly list the surfaces whose access model changed (cache management, admin nav, team/org management).

### 4. Existing Returner-only holders lose admin-portal access

**[File: apps/creative-portal/utils/isAdministrative.ts]**

**Function/Class:** isAdministrative

**Severity:** low

**Problem:** `isAdministrative` no longer treats `Returner` as conferring admin-portal access. A user who currently holds **only** the Returner role, with no admin flags and a non-admin (Creative) classification, will lose access to the admin portal after this ships. The role string itself is preserved on their record (hide-only), but it no longer grants anything.

**Impact:** Intended per the migration (returns move to order-admins), but it is a live behavioral change for existing data, not just a UI removal. If any real user relies solely on Returner for admin access today, they are cut off on deploy.

**Fix:** Confirm with the team that no active user depends solely on the Returner role for admin access (or that those users have been re-classified as order-admins). Behavioral, not a code defect — worth an explicit sign-off.

### 5. `ROLE_TO_JOB_TYPE` / `roleToJobType` duplicated verbatim across two files

**[File: apps/creative-portal/api/users/search/forAssignment/const.ts]**

**Function/Class:** ROLE_TO_JOB_TYPE, roleToJobType

**Severity:** low

**Problem:** This file and `api/mixtures/users/search/forAssignment/const.ts` are byte-identical, and this PR applies the same edit to both. Pre-existing duplication, not introduced here, but the divergence risk grows each time both must be edited in lockstep.

**Impact:** Future maintenance hazard — a change to one and not the other would go unnoticed by types (both compile independently).

**Fix:** Optional/out of scope: consolidate into one shared const (e.g. under a shared `forAssignment` util) imported by both search paths.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ | 1 failure: `api/orders/createNew/__tests__/jobTimings.test.ts` — "Hook timed out in 10000ms" (took 17.3s under parallel load). **Flaky/unrelated** — the file is not touched by this PR and passes in isolation (7/7, 815ms). Otherwise 2107/2108 creative-portal, 1327 shared, 336 customer, 265 wysiwyg all pass. |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces |
| `npx turbo run lint` | ✅ | 0 errors (`--max-warnings 0`) across all workspaces |
| `npx turbo run build` | ✅ | Clean, exit 0 |

---

## Tests

- ✅ New unit tests cover the changed logic: `getRoleNameBasedOnJob` (RETURN→Admin, QA order-admin split), `roleToJobType`/RETURN assessment skip (`mixtures/.../utils.test.ts`), `buildRolesForSubmit`, `resolveAdminFlags`/`isSystemAdmin`, `isAdministrative`, `TaskRoleDropdown` consts, `makeTeamMemberComparator`/`sortRolesByHierarchy`.
- ✅ Existing tests updated to drop the removed roles (guards, MainNav, TeamMemberModal hooks, jobs conflict toast, personal-info-content, mocks).
- ✅ Typecheck + lint + build all clean.
- ⚠️ The single test failure is a pre-existing flaky timeout in an unrelated file — confirmed passing in isolation. Not a regression from this PR.
- ❌ Manual + E2E testing checkboxes are unchecked. The two highest-risk items (Issues 1 & 2 — system-admin/flag access against the live OMS backend, and return-job assignment listing order-admins) can **only** be verified against the backend and are not covered by unit tests.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Code is internally consistent; correctness of admin gating depends on an unverified backend contract (Issues 1–2) |
| Regression risk | ⚠️ Medium — access-control change to system-admin/admin surfaces with the legacy fallback removed |
| Tests | ✅ Good unit coverage for the code-side logic; ⚠️ backend-dependent paths need manual verification |
| Code quality | ✅ Clean, well-commented, callers consistently updated |
| Validation suite | ⚠️ typecheck/lint/build pass; test has 1 flaky, unrelated failure (passes in isolation) |
| Mergeable state | ✅ GitHub reports `clean`; validation effectively green (sole failure is flaky/unrelated) |

---

## Recommendation

**Approve with suggestions** — contingent on the devtest verification below, which the PR itself lists as outstanding.

1. **Before merge, verify the backend contract (Issues 1 & 2).** In devtest, confirm a System Admin's auth response carries `classification` equal to `"System Admin"` (exact string/casing) **and** sets `userAdmin`/`orderAdmin`/`organizationAdmin`. This is the linchpin of the whole change and the removed code explicitly claimed classification was not returned. Attach evidence to the PR. If it does not hold, `isSystemAdmin`/`resolveAdminFlags` need a different signal.
2. **Complete the manual test plan** for the three named surfaces and the return-job assignment listing order-admins (the boxes left unchecked), since unit tests cannot cover them.
3. **Expand the PR description** (Issue 3) to reflect the full scope — enum retirement of all four roles and the flag/classification-based admin-gating migration.
4. **Get sign-off** that no active user relies solely on the Returner role for admin access (Issue 4).
5. **Optional:** de-duplicate `ROLE_TO_JOB_TYPE`/`roleToJobType` across the two search paths (Issue 5).
6. The single failing test is a flaky, unrelated timeout (`jobTimings.test.ts`, passes in isolation) — not a blocker for this PR.
