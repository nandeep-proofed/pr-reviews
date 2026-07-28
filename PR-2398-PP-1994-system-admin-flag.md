# PR Review: fix/PP-1994: gate system-admin surfaces on the systemAdmin flag

**PR:** https://github.com/Proofed/B2BWebserver/pull/2398
**Jira:** https://proofed.atlassian.net/browse/PP-1994
**Status:** Jira not fetched directly (Atlassian MCP not authenticated this session) — requirements below are reconstructed from Hideshi's guidance quoted in the PR body (comment 66686) and the PP-1890 context.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Do **not** use the classification *name* to determine access rights — that hardcodes FE logic and defeats the data-driven security config | Both remaining name comparisons removed. `isSystemAdmin` no longer reads `classification`; the non-admin tier is derived from §4.1 admin flags via `isNonAdminEntry`. `SYSTEM_ADMIN_CLASSIFICATION` and `NON_ADMIN_CLASSIFICATION` deleted with their `consts.ts`. | ✅ Addressed |
| Use the Rev 4.0 properties (privilege derived from classification in realtime) to determine privilege | `systemAdmin` flag added to `AdminFlagSource`/`AdminFlags`; `isSystemAdmin` returns `resolveAdminFlags(user).systemAdmin`. `flagGuard("systemAdmin")` now available for SSR gating. | ✅ Addressed |
| The classification name can change in future without the FE knowing | No fallback to the classification name kept anywhere. Anti-regression tests assert `isSystemAdmin({ classification: "System Admin" })` is `false` and `resolveAdminFlags` derives nothing from a classification/role. | ✅ Addressed |
| Privileged surface must fail safely | Fail-closed: a missing `systemAdmin` flag denies. Header item hidden, `DELETE /api/cacheManagement` returns 403. Verified by `removeAllCache.test.ts` (403 with every other flag, 403 for the "System Admin" classification without the flag, 403 with no session user). | ✅ Addressed |
| `systemAdmin` reaches the FE without extra plumbing | Added to `CreativeSessionUser` only (Authentication §5.1 returns it; Lookup/Search do not). Login route spreads the auth response straight into `req.session.user`, and each MFA/TOTP write spreads the existing user, so the flag survives. Confirmed in `packages/shared/api/login/index.ts`. | ✅ Addressed |

**Scope note:** the PR also folds the pre-existing inline non-admin rule in `TeamMemberModal/hooks.ts` into the shared `getNonAdminHierarchyIds` helper (de-duplication, no behavior change) and reworks `buildRolesDescription` to take the user object and gate on `isAdministrative`. Both are in-scope tidy-ups of the same "no name comparisons" theme, not scope creep.

---

## Architecture Analysis

The change completes the PP-1890 migration from classification-name gating to flag-based gating. `resolveAdminFlags` remains the single choke point; `systemAdmin` slots in beside the existing three flags, and `isSystemAdmin`/`isAdministrative`/`useAdminFlags`/`flagGuard` all compose from it rather than re-deriving. The two *display* surfaces that still compared `classification === "Creative"` (Roles cell, role filter) now share one `getNonAdminClassifications` helper built on `isNonAdminEntry`, and the modal's duplicate copy of that rule collapses into the same helper — so the name literal exists in exactly zero places after this PR.

The trust boundary is unchanged: both the client Header gate and the authoritative server 403 read the same object (`session.user` server-side, written only from the server-to-server OMS auth response into a signed iron-session cookie). The Header gate stays cosmetic; the 403 is the real guard. This is the correct shape for a privileged surface.

`DivisionalHierarchyEntry` (§4.1) carries only `orderAdmin`/`organizationAdmin` (confirmed in `types.ts`), so `isNonAdminEntry` checking exactly those two flags is correct for the data available, and the User type behind the personal-info Roles line (§8.2) does carry `orderAdmin`/`organizationAdmin`/`userAdmin`/`classification`/`roles`, so `isAdministrative(currentUserData)` resolves correctly there.

---

## Issues Found

### 1. Role filter can briefly show a flagless classification while the hierarchy is loading

**[File: apps/creative-portal/components/molecules/tables/TeamMembersTable/partials/RolesFilter/utils.ts]**

**Function/Class:** buildRoleFilterOptions

**Severity:** low

**Problem:** `buildRoleFilterOptions` filters classifications against `getNonAdminClassifications(hierarchy)`, which is empty until the §4.1 hierarchy resolves. The sibling Roles **cell** (`consts.tsx`) was given an explicit `!isHierarchyLoading` guard for exactly this window — "nothing is surfaced until the hierarchy has resolved" — but the filter has no equivalent guard. If a loaded member carries a flagless classification (e.g. "Creative") and the members query resolves before the hierarchy, that classification can appear as a filter option for a beat and then vanish once the hierarchy populates the set.

**Impact:** Purely transient/cosmetic — a filter option can flicker in and out during initial load. No incorrect access, no persisted state. The old name-based filter never had this window because `"Creative"` was known synchronously.

**Fix:** Either accept it (it self-corrects on the next render) or thread the hierarchy-loading flag through and skip building options until it resolves, mirroring the cell. Optional given the low impact.

### 2. `isNonAdminEntry` treats a userAdmin-only classification as non-admin

**[File: apps/creative-portal/api/divisionalHierarchy/utils.ts]**

**Function/Class:** isNonAdminEntry

**Severity:** low

**Problem:** The predicate is `!entry.orderAdmin && !entry.organizationAdmin`. A classification that conferred `userAdmin` but neither order- nor organization-admin would be classified non-admin and therefore hidden from the Roles cell/filter and required to hold a functional role in the modal.

**Impact:** None today — §4.1 (`DivisionalHierarchyEntry`) does not return `userAdmin`/`systemAdmin`, so no such entry can exist, and this exactly preserves the pre-existing modal behaviour. It only becomes wrong if §4.1 later starts returning `userAdmin`.

**Fix:** No change needed now — the code comment already documents the extension point ("Extend this predicate — not its call sites — if §4.1 starts returning `userAdmin`/`systemAdmin`"), which is the right place to catch it. Noting for the reviewer's awareness only.

### 3. Feature is gated on an unconfirmed BE field (author-flagged)

**[File: apps/creative-portal/utils/resolveAdminFlags.ts]**

**Function/Class:** isSystemAdmin

**Severity:** low (correctness of the fail direction) — but a functional blocker for the feature

**Problem:** The FE now reads `systemAdmin` from the Authentication (§5.1) response, which the PR description states has **not** been confirmed present in the live b2btest payload. If the BE does not return it, Refresh Cache stays hidden and the endpoint 403s for everyone.

**Impact:** Fail-closed, so there is no security risk — this is the correct failure direction for a privileged surface. But the feature is non-functional until the BE confirms the field. Existing sessions must also log out/in, since cookies written before this change do not carry `systemAdmin`.

**Fix:** No code change — this is a merge-coordination item. Confirm with the BE (Hideshi) that §5.1 returns `systemAdmin` on b2btest before/at merge, and complete the manual test (log in as a System Admin) that the PR checklist still has open.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ | creative-portal **2138/2138 pass** (all new tests included). One failure in `packages/shared/utils/formatWordQuantity.test.ts` — a locale number-grouping mismatch (`10,00,000` en-IN grouping vs expected `1,000,000`). Untouched by this PR (PR only adds `systemAdmin?: boolean` to a shared type); it's an environment `Intl` locale issue, not a regression. |
| `npx turbo run typecheck` | ✅ | 0 errors across all 5 workspaces. |
| `npx turbo run lint` | ✅ | 0 errors / 0 warnings across all workspaces. |
| `npx turbo run build` | ✅ | Clean — creative-portal + customer-portal both built, 0 type warnings. |

Note: the PR body reported a `jobTimings.test.ts` full-suite timeout; it passed in this run. The only failing test here is the unrelated `formatWordQuantity` locale case.

---

## Tests

- ✅ New `removeAllCache.test.ts` — covers the authoritative 403 path thoroughly (every-other-flag, "System Admin" classification without the flag, no session user, and success only for `systemAdmin: true`).
- ✅ New `Header/const.test.ts` — asserts System → Refresh Cache shows only for `isSystemAdmin` in the admin area, hidden otherwise.
- ✅ New `divisionalHierarchy/utils.test.ts` — the non-admin predicate keys off flags not names (renamed flagless entry still excluded; a "Creative"-named entry with a flag is kept).
- ✅ Updated suites carry anti-regression cases across `resolveAdminFlags`, `isAdministrative`, `useAdminFlags`, `guards`, `buildRolesDescription`, `buildRoleFilterOptions` — all assert the classification name never grants privilege.
- ✅ Meets the "every PR includes tests for new code" requirement.
- ⚠️ Manual end-to-end test (System Admin login on b2btest) still open — blocked on the BE §5.1 confirmation, per Issue 3.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ✅ |
| Code quality | ✅ |
| Validation suite | ⚠️ 1 unrelated pre-existing test failure (`formatWordQuantity`, locale); typecheck/lint/build clean, all PR tests pass |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Approve with suggestions** (pending the BE dependency)

1. **Confirm `systemAdmin` is returned by Authentication (§5.1) on b2btest before merging**, and complete the open manual test (System Admin login → Refresh Cache visible + `DELETE /api/cacheManagement` succeeds). The code is fail-closed, so nothing is unsafe, but the feature is inert until the BE field lands (Issue 3).
2. Optionally add the `!isHierarchyLoading` guard to `buildRoleFilterOptions` to match the Roles cell and avoid the transient filter-option flash (Issue 1). Low priority.
3. No action needed on `isNonAdminEntry` — the two-flag check is correct for §4.1 today and the extension point is already documented (Issue 2).
4. The single failing unit test (`formatWordQuantity` in `packages/shared`) is an environment/locale artifact unrelated to this PR — not a blocker for this change, but worth a separate look since it also fails on the base branch environment.
