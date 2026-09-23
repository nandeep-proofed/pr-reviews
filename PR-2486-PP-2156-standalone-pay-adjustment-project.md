# PR Review: feature/PP-2156: Accept a project on a standalone pay adjustment

**PR:** https://github.com/Proofed/B2BWebserver/pull/2486
**Jira:** https://proofed.atlassian.net/browse/PP-2156
**Status:** Reviewed at `187d1a230`. Updated 23 Sep 2026: every finding was re-verified against that head, the Jira ticket has since been read (the requirements table below matches it), and Issues 1 to 4 are fixed in `a4034ea91`. See the update immediately below.


---

## Update, 23 September 2026

### Fixes applied in `a4034ea91` (pushed to the PR branch)

- **Issue 1, resolved.** `findActiveOrganisationGroup` now also requires `organisationGroup.organizationId === organizationId`, so the cross-organization guarantee no longer rests on the OMS search being scoped correctly, and both new schema fields are `Yup.number().integer().positive().optional()`, so the `0` that dropped the scoping headers is rejected before the lookup. Covered by a new helper test (the list returns the id under a different organization) and two new schema tests. Mutation-checked: removing the new predicate line fails exactly the new ownership test.
- **Issue 2, resolved.** The route test is renamed to "skips the project lookup on a job-linked adjustment", with a comment recording that the schema rule it was named for does not run in that file.
- **Issue 3, resolved.** Added "returns the project when the active flag is absent", which pins the deliberate `active !== false` spelling.
- **Issue 4, resolved.** `organizationId` added to the child logger, and the success `.info` no longer repeats the fields `.with()` already carries.
- **Issues 5 and 6, left open deliberately.** Both touch the non-`.nullable()` plus `== null` pattern and the test-helper options that the standalone charge twin shares, so they belong in their own change rather than inside this PR.

### Checks run on the fix

| Check | Result |
| --- | --- |
| Tests, `api/compensations` + `api/utils/organisationGroups` | 68 passed (was 62) |
| `tsc --noEmit`, creative portal | clean |
| ESLint on the changed folders | 0 errors, 0 warnings |
| `next build` | not run, skipped by request |

### OMS persistence is formally deferred, not outstanding

The ticket settled this after the review was written:

- Hideshi (22 Sep): "As agreed, please create a change request. This will be scoped for Rev 4.03."
- Adam to QA (23 Sep): pass this through test "without the pay record being added against the Org Group", with a retest ticket once the API supports it.

So ticket requirement 1.2 and testing-note cases 1 and 3 are waived for this pass and are not a merge blocker. Note that this makes Issue 1 more important rather than less: Rev 4.03 is exactly when a wrong-client project would start being stored.

### Corrections to the review above

- **Jira was unreachable when the review ran and has since been read.** The requirements table, rebuilt from the PR description, matches the ticket. One nuance: the ticket asks only for validation against "an active, existing organization group", so the wrong-organization rejection behind Issue 1 comes from the PR description and the helper's docblock. It was a code-versus-documentation gap, not a missed requirement.
- `mergeable_state` now reports `clean`, not `unknown`.
- Still outstanding before merge: the validation suite (including the build) and `/security`.

---

## What this means for users (non-technical summary)

1. **The feature does not reach users yet, by design.** This is the API half only. Nobody can attach a project to a standalone pay adjustment through the app — the dropdown is PP-2157, and the backend (OMS) also still discards the project, which the author has documented and verified. Nothing in this PR is user-visible today.
2. **The "wrong client" safety check has a gap.** The point of this change is to refuse a pay adjustment pointed at a project that does not belong to the client it was filed under. That refusal currently leans entirely on the external system returning a correctly narrowed list, and one specific hand-typed value slips past it. No harm today because the project is discarded anyway — but once the backend starts storing it, internal time could be attributed to the wrong client, which is exactly what this check exists to prevent. Cheap to close now.
3. **Everything else about the rules works as advertised.** An adjustment with no project is still accepted; a project supplied without its client is refused; a project supplied on a job-linked adjustment is refused. All three were verified by running the real validation rules.

No other findings are user-visible. The remaining items are code health and test-coverage gaps.

---

## Jira Requirements vs Implementation

Requirements as restated in the PR description (see the caveat above).

| Requirement | PR Implementation | Status |
| --- | --- | --- |
| A standalone pay adjustment may carry a project (`organizationId` + `organizationGroupId`) | Both fields added to `createCompensationSchema.body` and to `CompensationCreate`; forwarded to OMS by `addCompensation` | ✅ Addressed |
| The project stays optional — an adjustment with no project is accepted exactly as today | `Yup.number().optional()` on both; verified by probe that omitting both passes | ✅ Addressed |
| The pair must be provided together | `project-needs-organization` test; verified firing under production validate options | ✅ Addressed |
| A project must not be set on a job-linked adjustment | `no-project-on-job-linked` test; verified firing under production validate options | ✅ Addressed |
| An unknown or deactivated project returns 400 and nothing is created | `findActiveOrganisationGroup` + `handleBadRequest`; the create is correctly short-circuited | ✅ Addressed |
| A project from the **wrong organization** returns 400 | Delegated entirely to the OMS search scoping; the local `.find()` never re-checks membership, and `organizationId: 0` drops the scoping headers | ⚠️ Partial — Issue 1 |
| Persist and return the project id and name | Not achievable — OMS accepts the POST but silently ignores both fields; documented and verified by the author on the test backend (record 5700) | ❌ Blocked upstream (correctly disclosed) |
| The modal's "Project (optional)" dropdown | Explicitly out of scope; PP-2157, which this blocks | ➖ Out of scope |

**Scope:** tight and single-purpose — 4 source files, 3 test files, no scope creep. Title and branch follow the convention.

---

## Architecture Analysis

The change sits entirely in the creative portal's BFF layer. `POST /api/compensations` gains two optional body fields and a pre-create validation hop, and a new app-local util `findActiveOrganisationGroup` wraps the existing `getOrganizationGroups` OMS client.

Three design choices are worth calling out, and all three check out:

- **Using the group *search* rather than the by-id lookup.** The docblock's rationale is that the by-id response carries no active flag. `OrganizationGroup.active` is indeed declared optional (`api/organizationGroup/types.ts:48`) and `fetchOrganisationGroupById` just types the raw response, so the repo neither confirms nor contradicts the claim — it rests on the author's backend verification. Worth recording that evidence in the ticket, because the whole design hangs on it (see Open Questions).
- **Mirroring the standalone charge payload.** Verified first-hand: `api/charges/schema.ts:34-68` carries the same `organizationId`/`organizationGroupId` pair with the same `no-mixed-payload` test and the same `== null` guard style. The new rules are a faithful copy of the established house pattern, including the deliberate difference that charge *requires* the pair when there is no order while pay leaves it optional.
- **Placement in `apps/creative-portal/api/utils/organisationGroups/`.** Correct — the customer portal has no compensations module, and the util depends on the creative-portal-only `api/organizationGroup` client, so it could not move to `packages/shared` without dragging that client along. The British/American spelling mix matches its pre-existing siblings exactly and introduces no new inconsistency.

One asymmetry the PR creates, worth a deliberate decision rather than a silent difference: a standalone **pay** adjustment's project is now verified active and owned, while a standalone **charge**'s is not verified at all — `api/charges/createAdjustmentCharge.ts:24-25` is just `await addCharge(requesterId, body)`. The new helper is a drop-in there.

---

## Issues Found

### 1. The cross-organization check has no local guard, and `organizationId: 0` removes the remote one (fixed in `a4034ea91`)

**[File: apps/creative-portal/api/utils/organisationGroups/findActiveOrganisationGroup.ts:43]**

> **In plain terms:** This change exists so an adjustment cannot be filed against a project belonging to a different client. That refusal currently depends entirely on the external system handing back a correctly narrowed list — nothing on our side double-checks that the project actually belongs to the client named on the request. One specific value typed into the request removes the narrowing altogether, and a project from any other client is then accepted. No damage today, because the external system throws the project away. The moment it starts keeping it, internal time can be attributed to the wrong client — the exact outcome this check was added to prevent.

**Function/Class:** findActiveOrganisationGroup

**Severity:** medium

**Confidence:** high (mechanism verified by probe; one link inferred — see Evidence)

**Steps to reproduce:**

1. Log in to the creative portal as any user with a live session (the route sets `requiredRoles: []`, so no specific role is needed).
2. From the DevTools console, `POST /api/compensations` with a valid standalone pay body plus `organizationId: 0` and `organizationGroupId: <the id of an active project belonging to a different client>`.
3. **Expected:** 400 "The selected project does not exist or is not active" — the project does not belong to organization 0.
4. **Actual:** the project-ownership check passes and the create proceeds, forwarding `organizationId: 0` with a foreign project id to OMS.

**Problem:** Two gaps compound. The `.find()` predicate checks the id and the active flag but never the organization:

```typescript
return organisationGroups.find(
  (organisationGroup) =>
    organisationGroup.id === organizationGroupId &&
    organisationGroup.active !== false
);
```

So the entire cross-organization guarantee is delegated to the search scoping — which `getOrganizationGroups` drops for any falsy `searchValue`:

```typescript
// apps/creative-portal/api/organizationGroup/index.ts:21
if (searchBy && searchValue) {
  headers.searchBy = searchBy;
  headers.searchValue = searchValue;
}
```

`organizationId: 0` reaches that line because the schema has no `.positive()`/`.integer()` and both new object-level tests use `== null` (`0 == null` is `false`), and because the route's guard is `organizationId != null`.

**Evidence:** All three links verified against the real code, and two adversarial probes run against the repo's own yup 0.32.11 with production's validate options (`{ abortEarly: true }`, from `packages/shared/api/utils/middlewares/withValidateRequestSchema.ts:91-100`):

- Schema probe: `organizationId: 0` + `organizationGroupId: 777` → `PASS`, `validatedData.body.organizationId === 0`. (`-5` and `1.5` also pass, but stay truthy so they remain scoped — `0` is the unique bypass value.)
- Header probe, using the header-building logic copied verbatim from `organizationGroup/index.ts:17-31`: `organizationId=143` → `{requesterId, searchBy:"orgId", searchValue:143, filterBy:"Both", Status:"A"}`; `organizationId=0` → `{requesterId, filterBy:"Both", Status:"A"}`. Both scoping headers gone.
- In-repo corroboration that an unscoped search returns other organizations' groups: `api/mixtures/partnerProjects/utils.ts:24-45` deliberately omits `searchBy`/`searchValue`, names the result `allGroups`, and then applies the very filter this helper lacks — `allGroups.filter((group) => group.organizationId === org.id)` under the comment "If the org is inactive, only keep groups belonging to it".
- **Inferred, not observed:** whether OMS answers an unscoped search with all organizations' groups (→ wrong 200) or rejects it (→ upstream error). The `fetchPartnerProjects` precedent is strong evidence for the former, but it is inference. Either way the intended 400 is not what happens.

**Impact:** Today: nil in data terms — the PR documents that OMS discards both fields, and nothing is disclosed to the caller (only the created compensation is returned). Structurally: the feature's headline guarantee has exactly one point of failure, and it sits outside this codebase — in the same OMS the PR body documents as ignoring these very field names. It also becomes a live mis-attribution bug the moment OMS implements persistence, which is the whole point of the ticket. No test in the PR can catch it: `findActiveOrganisationGroup.test.ts` mocks `getOrganizationGroups` wholesale, `createCompensation.test.ts` mocks the helper, and `schema.test.ts` never tries a non-positive id.

**Fix:** Two one-liners — defend locally, and stop `0` at the door:

```typescript
// findActiveOrganisationGroup.ts — don't rely on the remote scoping alone.
// organizationId is a required field on OrganizationGroup, so this is free.
return organisationGroups.find(
  (organisationGroup) =>
    organisationGroup.id === organizationGroupId &&
    organisationGroup.organizationId === organizationId &&
    organisationGroup.active !== false
);
```

```typescript
// schema.ts — ids are positive integers; 0 must never reach the lookup.
organizationId: Yup.number().integer().positive().optional(),
organizationGroupId: Yup.number().integer().positive().optional(),
```

Add a `findActiveOrganisationGroup` test where the returned list contains the id under a *different* `organizationId`, asserting `undefined`.

### 2. A test named for job-linked behaviour cannot observe it (fixed in `a4034ea91`)

**[File: apps/creative-portal/api/compensations/createCompensation.test.ts:137]**

> **In plain terms:** One of the new automated checks is named after the rule "a job-linked adjustment takes its project from the order", but it cannot actually see that rule — the part of the code that enforces it is switched off inside this test. The rule itself is genuinely covered elsewhere, so nothing is unprotected; the risk is a reviewer or future maintainer trusting a check that proves less than its name claims.

**Function/Class:** describe("createCompensation: project attribution") — it("leaves a job-linked adjustment to take its project from the order")

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible. The route's condition at `createCompensation.ts:37` is `organizationGroupId != null && organizationId != null` — `jobId` never enters it. This test and the preceding one (`:126`, "creates an adjustment with no project") both pass both project fields as `undefined`, so they traverse the identical branch for the identical reason. The rule that makes the test's name true is the schema's `no-project-on-job-linked` test, and this file mocks `withApiMiddleware` to a pass-through (`:13-15`) and hand-builds `validatedData.body` (`:68`), so the schema never runs here.

**Problem:** The test is redundant with `:126` and its name asserts coverage it does not provide.

**Impact:** Misleading coverage signal only. The rule is properly covered at `schema.test.ts:168`, so there is no actual gap.

**Fix:** Delete it, or rename it to what it can prove (e.g. "skips the project lookup when no project is supplied, even with a jobId").

### 3. The `active !== false` branch is unpinned — a future "tightening" would break live projects silently (fixed in `a4034ea91`)

**[File: apps/creative-portal/api/utils/organisationGroups/findActiveOrganisationGroup.ts:46]**

> **In plain terms:** The code deliberately treats a project as usable when the external system does not say either way about whether it is switched on. That deliberate choice is not protected by any automated check, so a future tidy-up could quietly flip it — and then valid, live projects would start being rejected with a misleading "project does not exist or is not active" message.

**Function/Class:** findActiveOrganisationGroup

**Severity:** low

**Confidence:** high (mutation-verified)

**How to spot it:** Code health, not user-reproducible today. `OrganizationGroup.active` is optional (`api/organizationGroup/types.ts:48`) and the predicate deliberately uses `active !== false`, i.e. "flag absent means active". Mutating that line to `active === true` leaves the entire test suite green: `findActiveOrganisationGroup.test.ts:42` supplies `active: true` and `:57` supplies `active: false`, but nothing supplies `undefined`.

**Problem:** The semantically interesting case — the one the `!== false` spelling exists for — has no test.

**Impact:** A later refactor toward the more natural-looking `active === true` would start rejecting live projects whose OMS row omits the flag, surfacing as a spurious 400, with no test to stop it.

**Fix:** Add one case to `findActiveOrganisationGroup.test.ts`:

```typescript
it("accepts a project whose active flag is absent", async () => {
  mockGetOrganizationGroups.mockResolvedValue([
    { id: 407, name: "4.08.2025" }
  ]);

  await expect(find(407)).resolves.toMatchObject({ id: 407 });
});
```

### 4. The rejected project pair cannot be reconstructed from the logs (fixed in `a4034ea91`)

**[File: apps/creative-portal/api/compensations/createCompensation.ts:28]**

> **In plain terms:** When the new check refuses a project, the diagnostic record keeps the project but not the client it was checked against. Since the refusal is precisely about those two not matching, anyone investigating a complaint sees half the story.

**Function/Class:** createCompensation

**Severity:** low

**Confidence:** high

**How to spot it:** Code health. The new child logger is built with `{ jobId, organizationGroupId, proofedUserId }` — `organizationId` is absent — and it is the logger handed to `handleBadRequest` when the new 400 fires at `:45-51`.

**Problem:** The 400 exists to report an organization/project mismatch, but only one side of the pair is logged.

**Impact:** Support and Sentry triage of "my project was rejected" cannot tell which organization it was checked against.

**Fix:** Add `organizationId` to the `.with({ ... })` call. While there, the later `.info(msg, { jobId, proofedUserId, amount })` at `:58-61` re-sends two fields already merged by `.with()` — only `amount` adds information, so the duplicates can be trimmed.

### 5. An explicit JSON `null` is rejected, and `jobId: null` reports the wrong rule

**[File: apps/creative-portal/api/compensations/schema.ts:37]**

> **In plain terms:** If a caller sends the project fields as an explicit "empty" value rather than leaving them out, the request is refused instead of being treated as "no project" — and in one combination the refusal quotes the wrong reason, which would send someone debugging in the wrong direction. Nothing in the app does this today, and nothing bad is ever saved; it is a trap laid for whoever wires up the dropdown next.

**Function/Class:** createCompensationSchema

**Severity:** low

**Confidence:** high (probe-verified against the real yup with production's validate options)

**How to spot it:** Not reachable from the product. `buildCompensationPayload` (`components/organisms/modals/AdjustPayOrChargeModal/utils.ts:184-199`) emits `undefined`, which axios' default JSON serialization drops, and `CompensationCreate` types these fields without `| null`. It *is* reachable by an authenticated operator hand-crafting a request, and the outcome is always a 400 — nothing is created.

**Problem:** `Yup.number().optional()` is not `.nullable()`, so yup casts `null` to `NaN`:

- `{ organizationId: null, organizationGroupId: null }` → 400 typeError, `body.organizationGroupId must be a \`number\` type, but the final value was: \`NaN\``. (Omitting both still passes correctly — probe-verified — so the schema's "an adjustment with no project is still accepted" comment holds for the shape the wire actually carries.)
- `{ jobId: null, organizationId: 143, organizationGroupId: 407 }` → the object-level test runs on the **cast** value, `NaN == null` is `false`, so a null `jobId` is misread as job-linked and the response is the misleading "When jobId is provided, organizationId and organizationGroupId must not be set".

**Impact:** Misleading error message in one unreachable combination. Worth noting honestly: `jobId: null` already 400'd on `develop` with the same NaN typeError, so this PR changes no accept/reject outcome — only the message, in that one case. The same non-`.nullable()` + `== null` pattern is the pre-existing house style in the charge twin (`api/charges/schema.ts:37-56`), so this is not a defect introduced here.

**Fix:** Opportunistic hardening — add `.nullable()` to the three number fields (or transform `null` to `undefined`) so the `== null` guards mean what they read as. If you'd rather keep the house pattern consistent, changing the charge twin at the same time is the cleaner call.

### 6. The new schema tests run under validate options production never uses

**[File: apps/creative-portal/api/compensations/schema.test.ts:131]**

> **In plain terms:** The new automated checks for the validation rules are run with different settings than the live server uses. The rules themselves do behave the same under both, so nothing is currently mis-protected — but the checks aren't exercising the live configuration, and one of the differences is exactly what produces the wrong error message described above.

**Function/Class:** validateBody

**Severity:** low

**Confidence:** high

**How to spot it:** Code health. The helper validates with `{ abortEarly: false, stripUnknown: true }`, while production uses `{ abortEarly: true }` and nothing else (`packages/shared/api/utils/middlewares/withValidateRequestSchema.ts:91-100`).

**Problem:** Two divergences. `abortEarly` changes which error surfaces first — the ordering that produces Issue 5's wrong message is only observable under the production setting. And `stripUnknown` is never passed in production, so unknown body keys survive into `validatedData.body` and are POSTed verbatim to OMS; the tests assert a stripped shape the route never produces.

**Impact:** Both new rules were independently verified to fire correctly under production options, so there is no live gap. The tests simply do not cover the configuration that runs. The unknown-key pass-through is pre-existing middleware behaviour, not introduced by this PR.

**Fix:** Validate with `{ abortEarly: true }` in `validateBody` to match production. If the stripping behaviour is actually wanted, that is a separate change to `withValidateRequestSchema`.

---

## Suggestions (non-blocking)

- **Close the charge/pay asymmetry, or say why not.** `api/charges/createAdjustmentCharge.ts:24-25` performs no project validation at all, so a standalone charge can name a deactivated or foreign project freely. The new helper is a drop-in. Either note the exemption in the ticket or file the follow-up.
- **Extract the "list an organization's projects" call.** `findActiveOrganisationGroup.ts:36-42` and `api/mixtures/partnerProjects/utils.ts:32-38` now encode the same `getOrganizationGroups` argument shape. A shared `listOrganizationProjects({ requesterId, organizationId, status })` would hold it once. (Roughly six lines — judgement call whether it earns the indirection yet.)
- **Home the two new constants next to their peers.** `ACTIVE_STATUS` / `ALL_GROUP_TYPES` would sit naturally in `api/organizationGroup/consts.ts` beside `OrganizationGroupSearchByTerm`, and `GetOrganizationGroupsParams.filterBy`/`status` could be narrowed off bare `string`. Context: there are 11 inline `"Both"` literals across the app and `apps/auth-provider` uses lowercase `"both"`, so the repo is already inconsistent — the named consts here are an improvement, just not shared ones.
- **Cover the failure path of the lookup.** Nothing anywhere asserts what happens when `getOrganizationGroups` rejects. An OMS 404 on the group *search* currently becomes the status of `POST /api/compensations` via `handleEndpointError`, which is fail-closed (correct) but opaque (a 404 from a POST that exists).
- **`describe` separator.** The two new blocks use `"X: Y"` while the pre-existing blocks in the same file and the sibling api tests use `"X — Y"`. The head commit changed this deliberately, so either finish the file or revert to house style. No CLAUDE.md rule governs it.

---

## Open Questions

Unconfirmed — these are questions for the author, not defects.

- The design rests on "the by-id response carries no active flag at all". The repo can't confirm it (`OrganizationGroup.active` is merely optional, and `fetchOrganisationGroupById` types the raw response) — could the backend verification be recorded in the ticket? If by-id *does* return `active`, a single-request `fetchOrganisationGroupById` + `organizationId`/`active` check replaces the whole-org list fetch. — `apps/creative-portal/api/utils/organisationGroups/findActiveOrganisationGroup.ts:17`
- Does the OMS group search return every organization's groups when `searchBy`/`searchValue` are absent, or does it reject the call? This decides whether Issue 1's endpoint outcome is a wrong 200 or an upstream error. — `apps/creative-portal/api/organizationGroup/index.ts:21`
- The project picker requests groups with **no** Status header (`AdjustPayOrChargeModal/hooks.ts:77` → `api/mixtures/partnerProjects/utils.ts:37`, where `status` is `undefined` and the header is therefore omitted), while this validator hardcodes `status: "A"`. Can a deactivated project appear in the dropdown and then 400 on submit? — `apps/creative-portal/api/utils/organisationGroups/findActiveOrganisationGroup.ts:39`
- `filterBy: "Both"` includes virtual groups, and the current charge picker does list them unfiltered (`AdjustPayOrChargeModal/hooks.ts:93-104`). But every other project list in the app excludes them (`CustomerTable/useOrganzationsFilter.ts:44-50`, `DeactivatedProjectsModal/hooks.ts:19`, `pages/partners/utils.ts:20`). If PP-2157's Pay picker follows the `virtual === false` convention, is `"Both"` here too permissive? — `apps/creative-portal/api/utils/organisationGroups/findActiveOrganisationGroup.ts:8`
- Confirming the sequencing: PP-2157 carries both the dropdown *and* the `buildCompensationPayload` change that actually sends the fields? Nothing on this branch populates them, so the new path is dead from the UI. — `apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/utils.ts:184`
- Pre-existing and out of scope, noted only because it came up while tracing reachability: this route's middleware options are `{ schema }` only — no CSRF, and `requiredRoles: []`, so any live session reaches it. The charge twin is identical, and no creative-portal route uses `requiredRoles`. Is that a known accepted position?

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ | Not run in this review |
| `npx turbo run typecheck` | ⏭️ | Not run in this review |
| `npx turbo run lint` | ⏭️ | Not run in this review |
| `npx turbo run build` | ⏭️ | Not run in this review |

The suite was **not** run as part of this review, so nothing here attests to it. Run it scoped to the only workspace this PR touches before merging:

```bash
npx turbo run test typecheck lint build --filter=@proofed/creative-portal
```

Two partial data points gathered incidentally while verifying findings, in a throwaway worktree at the PR head — not a substitute for the suite: the PR's own 25 tests passed, and `prettier --check` and `eslint` were clean on all 7 changed files. The author reports typecheck and lint passing with 0 warnings; the PR checklist leaves "Build successful before PR" unticked.

---

## Tests

- ✅ Every changed source file has a corresponding test file — the project's "tests required" rule is met.
- ✅ The rules are genuinely guarded, not just nominally covered: removing both object-level `.test()` blocks from `schema.ts` fails exactly the three rejection tests, and removing the route's guard block fails 2 of the 4 route tests.
- ✅ `schema.test.ts` covers all six pair/job-linked permutations, and correctly documents why it runs against real yup (`packages/shared/vitest.config.ts:37` does alias yup to a cast-only mock).
- ✅ `findActiveOrganisationGroup.test.ts` pins the exact outbound search arguments — a good choice, since that argument shape is the whole contract.
- ⚠️ The `organizationId: 0` / cross-organization path is invisible to every test in the PR (Issue 1): the helper's test mocks `getOrganizationGroups`, the route's test mocks the helper, and the schema's test never tries a non-positive id.
- ⚠️ `active: undefined` is untested and the branch is mutation-provable as unpinned (Issue 3).
- ⚠️ One route test cannot observe the behaviour it is named for and duplicates its neighbour (Issue 2).
- ⚠️ No test anywhere covers the lookup or the create *rejecting* — `handleEndpointError` is mocked to a bare `vi.fn()` and never asserted.
- ⚠️ The schema tests run under non-production validate options (Issue 6).
- ℹ️ `handleBadRequest` is mocked with a comment claiming the real helper needs a Logtail logger; it does not — `getCorrIdFromLogger` is fully null-safe. The real blocker is only that the fake logger lacks `.error`. Adding `error: vi.fn()` and dropping the mock keeps the tests green *and* covers the real 400 status and `{ error: { message } }` body shape, which nothing currently asserts.
- ℹ️ Mock fixtures are 3-key partials of a 19-required-field interface with no casts. That typechecks only because `vi.mock` factory returns are unchecked — fine here, but it means an OMS-side rename of `active` would not be caught by these tests.

### Suggested manual QA script

The feature has no UI surface yet, so this is a DevTools-console script against the PR branch, logged in to the creative portal. `POST /api/compensations` with a valid standalone pay body plus:

1. No project fields → expect **200**.
2. An active project with its own organization → expect **200**. (The project is not persisted — OMS discards it; confirm via Compensation Search that no project comes back, matching the PR's disclosure.)
3. `organizationGroupId` only → expect **400**, "must be provided together".
4. `organizationId` only → expect **400**, "must be provided together".
5. `jobId` plus both project fields → expect **400**, "must not be set".
6. A project belonging to a different organization → expect **400**, "does not exist or is not active".
7. A non-existent project id → expect **400**.
8. A deactivated project id → expect **400**.
9. **Issue 1 check:** `organizationId: 0` plus an active project id from *another* organization → **should** be 400; expect it to currently pass the check. This is the regression test for the fix.
10. **Issue 5 check:** `jobId: null` plus a valid project pair → note the message names the job-linked rule rather than the null `jobId`.

Cases 1-8 correspond to the author's own manual test table; 9 and 10 are the additions from this review.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ One gap in the feature's core guarantee (Issue 1); everything else verified correct |
| Regression risk | ✅ Low — the only caller never sends the new fields, both new fields are additive optionals, and no existing payload can trip the new rules |
| Tests | ⚠️ Present, meaningful and mutation-verified, but blind to Issue 1 and to three other branches |
| Accessibility | ➖ n/a — API only, no UI |
| Error handling | ✅ Fail-closed throughout; the create is correctly short-circuited. ⚠️ Minor: logging omits `organizationId` (Issue 4) and an upstream lookup failure surfaces opaquely |
| Security | ⚠️ No new vulnerability. Inputs are validated; nothing is disclosed to the caller; no secrets or PII logged. `/security` is still required per CLAUDE.md before merge — this was a breadth pass, and it should look at Issue 1 and at the pre-existing no-CSRF / `requiredRoles: []` position on this route |
| Code quality | ✅ Clean, well-commented, conventions followed (interface ordering, descriptive names, placement, prettier/eslint) |
| Validation suite | ⏭️ Not run — must be run before merge |
| Mergeable state | ⚠️ GitHub reports `mergeable_state: "unknown"`; validation not run, so this review cannot attest to it |

---

## Recommendation

> Updated 23 Sep 2026: step 1 is done, and Issues 2 to 4 of step 5 with it (see the update at the top). Steps 2 and 3 still stand. Step 4's OMS persistence question has been answered: the client deferred it to OMS Rev 4.03.

**Approve with suggestions** — contingent on the validation suite passing, which this review did not run.

This is careful, well-documented work. The rules do what the PR says they do (verified by running them), the scope is tight, the tests are real rather than decorative, and the upstream OMS limitation is disclosed honestly rather than papered over. Nothing here is a blocker.

Before merge:

1. **Fix Issue 1** — add `organisationGroup.organizationId === organizationId` to the `.find()` predicate and `.integer().positive()` to both new schema fields, plus a test where the list contains the id under a different organization. Two lines and a test; it closes the one gap in the feature's stated purpose and matters most exactly when OMS starts persisting the fields.
2. **Run the validation suite** — `npx turbo run test typecheck lint build --filter=@proofed/creative-portal`. The PR's "Build successful before PR" box is unticked.
3. **Run `/security`** — mandated by CLAUDE.md and unticked on the PR.
4. **Answer the Open Questions in the ticket**, particularly the by-id/`active` evidence (the design rests on it) and whether virtual groups count as projects here (it shapes PP-2157).
5. Tidy Issues 2-6 opportunistically — all small, none blocking.

Also worth deciding deliberately rather than by omission: the new charge/pay validation asymmetry, and whether PP-2157 carries the payload wiring so the feature is not left dead from the UI.
