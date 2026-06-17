# PR Review: PP-1754: Upgrade TypeScript to v6 and development tooling

**PR:** https://github.com/Proofed/B2BWebserver/pull/2261
**Jira:** https://proofed.atlassian.net/browse/PP-1754
**Status:** Code Review
**Head SHA reviewed:** `e4460091f` (includes a fresh merge of `develop` → branch + the file-type build fix)
**Scope:** 103 files, +2,077 / −1,892, 49 commits

> **Update (`e4460091f`):** The build blocker (Issue 1) has been **fixed** — `packages/shared` file-type pinned to `19.6.0`; `npx turbo run build` now passes **4/4 workspaces**. Issues 2 (Sentry `transpileClientSDK`) and 3 (Storybook alpha) are **intentionally deferred** to separate tickets per team decision — not blockers for this PR. See the updated Validation Checks, Summary, and Recommendation.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Upgrade Turborepo, Playwright, TypeScript to latest compatible | Turbo `2.3.3 → 2.9.6`, Playwright `1.45.1 → 1.59.1`, TypeScript `5.3.3 → 6.0.2` across all workspaces | ✅ Addressed |
| 2. Storybook: remove alpha in creative portal + align versions across portals | creative-portal still pins `@storybook/* ^7.0.0-alpha.48`; storybook app on `^7.6.17` — **intentionally deferred to a separate ticket** | ⏸️ Deferred (by decision) |
| 3. Sentry: disable unnecessary client SDK transpilation | `packages/shared/scripts/nextConfig.js:57` still `transpileClientSDK: true` — **intentionally deferred** (note: PR description should be corrected, it claims `false`) | ⏸️ Deferred (by decision) |
| 4. Resolve `@types/node` version mismatch across packages | Both apps pin `@types/node: ^20` (was `18.7.6` in creative) | ✅ Addressed |
| 5. All packages must build and test successfully | Build now **passes 4/4** after the file-type fix (`e4460091f`); shared has 1 locale-driven test failure (env, not code) | ✅ Addressed (post-fix) |

**Scope note:** The PR is otherwise well-contained to tooling/config/type changes. It does pull in two functional package upgrades (swiper 8→11, react-toastify 9→10), a file-type rework, and an iron-session v6→v8 auth-library upgrade — all justified by the bundler-resolution change but they expand the blast radius and require manual QA.

---

## Architecture Analysis

The core change is sound and well-reasoned: `moduleResolution: "node" → "bundler"` (the modern Next.js default), `baseUrl` removed in favour of explicit `paths`, explicit `types` arrays, and `target ES2015 → ES2018`. The migrations are mechanical and consistent:

- **react-toastify 9→10** (`Toast` atoms/molecules, wysiwyg Toast): `toastify.TYPE.*` enums → string literals (`"success"`, `"error"`, …) and `toast.POSITION.BOTTOM_LEFT → "bottom-left"`. Correct for v10's API. ✅
- **swiper 8→11** (`OrderJobs`): `Mousewheel` moved from `"swiper"` → `"swiper/modules"`. Correct for v11. ✅ (needs visual QA — 3 majors.)
- **file-type rework** (`processWorkItemContentWithMetadata`): `fileTypeFromFile(path)` → header-only read (`openSync`/`readSync` first 4100 bytes) + `fileTypeFromBuffer`. The header-only approach is actually a memory improvement over the whole-file read described in the PR body. ✅ logic — but see Issue 1 for the packaging fallout.
- **iron-session 6→8**: full API migration (`withIronSessionApiRoute`/`withIronSessionSsr` → `getIronSession`); see Issue 5 for the breakdown and the deploy-time verifications.
- **tsconfig `paths`** migration is complete (typecheck passes across all 5 workspaces, proving every alias import resolves).
- **husky 8→9** (`7d9fd501b`): `prepare: "husky install"` → `"husky"`, and `.husky/pre-commit` drops the now-deprecated v8 boilerplate (`#!/usr/bin/env sh` + `. "$(dirname -- "$0")/_/husky.sh"`), leaving just `yarn lint-staged`. Correct for v9 — the sourcing line is deprecated in v9 and removed in v10; the hook still runs lint-staged. ✅

The bundler-resolution change is the right call but has a sharp edge: it now resolves dependencies to their raw ESM `source` entry, which surfaces un-transpiled modern syntax to Next's webpack — exactly what broke the customer-portal build (Issue 1, now fixed).

---

## Issues Found

### 1. customer-portal production build fails — file-type `v`-flag regex ✅ FIXED (`e4460091f`)

**[File: packages/shared/package.json]**

**Function/Class:** `file-type` dependency (`^22.0.1`) consumed by `packages/shared/utils/files/processWorkItemContentWithMetadata.ts`

**Severity:** high — **resolved**

**Resolution:** Pinned `packages/shared` file-type `^22.0.1 → 19.6.0` in commit `e4460091f`. `npx turbo run build` now passes **4/4 workspaces** (customer-portal + creative-portal both green). The sole consumer uses only `fileTypeFromBuffer`, which is identical across 19.x/22.x, so no functionality is lost; the version split (Issue 4) is also resolved. Original analysis retained below for context.

**Problem:** `npx turbo run build` failed on `@proofed/customer-portal`:

```
../../packages/shared/node_modules/file-type/source/index.js
Module parse failed: Invalid regular expression flag (1171:8)
> if (/^\d+$/v.test(version) && version >= 1000 && version <= 1050) {
Import trace: processWorkItemContentWithMetadata.ts → fileUploadStream.ts
            → pages/api/orders/createOrder/stream.ts
```

`packages/shared` pinned `file-type: ^22.0.1` (resolves to 22.0.1), whose source uses the ES2024 `v` (unicodeSets) regex flag. Under the new `moduleResolution: bundler`, the import resolves to file-type's raw ESM `source/index.js`, which Next's webpack/acorn parser cannot handle. creative-portal builds fine; customer-portal pulls this shared code path through an API route and failed.

**Impact:** Was a hard blocker — customer-portal could not be built/deployed. This was the PR's own change (file-type `^22.0.1` added in commit `02be61c37`, "Address PR review side-effects items"), not a side effect of the develop merge.

**Fix (applied):** Aligned shared's file-type to the same major the customer-portal already pins (`19.6.0`) — file-type 19.x does **not** use the `v` flag and resolves cleanly. This also removed the version split (Issue 4):

```jsonc
// packages/shared/package.json
"file-type": "19.6.0",   // was "^22.0.1"
```

(Alternative if 22.x had been required: add file-type to `transpilePackages` in customer-portal's next config so webpack down-levels the `v` flag — heavier and would leave the version split in place.)

### 2. Sentry `transpileClientSDK` still enabled — Jira requirement #3 ⏸️ DEFERRED (by decision)

**[File: packages/shared/scripts/nextConfig.js]**

**Function/Class:** Sentry webpack options (line 57)

**Severity:** low (deferred to a separate ticket per team decision)

**Note:** Deferred — not a blocker for this PR. One housekeeping item remains: the PR description claims `transpileClientSDK: false` while the code is still `true`; correct the description so it doesn't imply work that was deferred.

**Detail:** Jira req #3 asks to "disable unnecessary client SDK transpilation," and the PR description explicitly claims `transpileClientSDK: false`. The actual value is still `transpileClientSDK: true`, and the PR diff does not touch this line. `transpileClientSDK: true` transpiles the Sentry SDK for IE11, inflating the client bundle (the project targets ES2018+).

**Fix (when the deferred ticket is picked up):**

```js
// packages/shared/scripts/nextConfig.js
transpileClientSDK: false,
```

Then re-verify Sentry still initializes and reports.

### 3. Storybook alpha not removed / versions not aligned — Jira requirement #2 ⏸️ DEFERRED (by decision)

**[File: apps/creative-portal/package.json]**

**Function/Class:** `@storybook/*` devDependencies

**Severity:** low (deferred to a separate ticket per team decision)

**Detail:** creative-portal still pins `@storybook/addon-*`, `@storybook/nextjs`, `@storybook/react`, and `storybook` at `^7.0.0-alpha.48`, while the storybook app uses `^7.6.17`. Jira req #2 asks to remove the alpha and align versions across portals. The PR body lists "Storybook 8 — out of scope"; per team decision the alpha alignment is deferred to its own ticket.

**Fix (when the deferred ticket is picked up):** Bump creative-portal's `@storybook/*` and `storybook` to the stable `^7.6.17` line used by the storybook app, and update the Jira scope so req #2 is formally tracked rather than silently dropped.

### 4. file-type version split across the monorepo ✅ FIXED (`e4460091f`)

**[File: packages/shared/package.json + apps/customer-portal/package.json]**

**Function/Class:** `file-type` dependency

**Severity:** low — **resolved**

**Problem:** `packages/shared` declared `file-type: ^22.0.1` while `apps/customer-portal` declares `file-type: 19.6.0` — two majors of the same library, with the shared code path (the only real consumer) resolving to 22.x. This was the root cause of Issue 1 and a maintenance smell.

**Resolution:** Both now resolve to `19.6.0` after the Issue 1 fix. Note: `apps/customer-portal` still *declares* `file-type: 19.6.0` but imports it nowhere (dead dependency, re-introduced by the develop merge — commit `02be61c37` had originally removed it). Optional follow-up: drop the unused line from `apps/customer-portal/package.json`. Left in place for now per author preference.

### 5. iron-session v6→v8 migration — code correct, but PR description wrong + 2 deploy-time verifications

**[File: packages/shared/lib/session.ts, packages/shared/api/utils/middlewares/withApiMiddleware/withApiMiddleware.ts, `@types/session.ts` (×3), `withUserProvided` enhancers, page/api session call sites]**

**Function/Class:** iron-session session handling

**Severity:** low (migration is correct; items below are a description fix + pre-deploy verification, not code defects)

**Context:** Despite the PR description stating *"No iron-session v8 upgrade (not needed once moduleResolution changed)"*, the PR **does** upgrade iron-session `^6.1.3 → ^8.0.4` across creative-portal, customer-portal, and shared (commit `12e194176`, "Upgrade iron-session from v6 to v8"). The v6→v8 API migration is done properly:

| v6 (before) | v8 (after) | Correct? |
|---|---|---|
| `IronSessionOptions` | `SessionOptions` (renamed in v8) | ✅ |
| `withIronSessionApiRoute(handler, opts)` from `iron-session/next` | manual `req.session = await getIronSession(req, res, opts)` wrapper in `withApiMiddleware` | ✅ |
| `withIronSessionSsr` | manual `getIronSession(ctx.req, ctx.res, opts)` + `ctx.req.session = session` in `withUserProvided` enhancers | ✅ |
| auto-augmented `http.IncomingMessage.session` | manually re-added (v8 dropped the automatic augmentation) | ✅ |
| all page/api call sites | use v8 `getIronSession<IronSessionData>(req, res, sessionOptions)` | ✅ |

All `iron-session/next` / `withIronSessionApiRoute` / `withIronSessionSsr` imports are gone, and **typecheck passes 5/5**, so the type side is sound. The runtime assignment (`req.session = await getIronSession(...)`) is present everywhere the augmented `req.session` type is read. **Mechanically, this is a clean, correct migration.**

**Action items (not code defects):**

1. **Correct the PR description** — it denies the v8 upgrade that the code actually performs. Reviewers/QA need to know a major auth-library bump is in scope.

2. **Verify before deploy — existing sessions may be invalidated.** iron-session changed its cookie seal format between v6 (`@hapi/iron`) and v7+/v8; a v6-sealed cookie generally cannot be unsealed by v8, so already-logged-in users will likely be **forced to re-login once** after deploy. Not data loss, but call it out in release notes and QA (log in on current prod → deploy → confirm graceful re-login, not an error).

3. **Verify before deploy — `password` length.** v8 hard-requires `SECRET_COOKIE_PASSWORD` ≥ 32 chars and throws at the first `getIronSession` call otherwise. v6 had the same minimum (so prod is *probably* fine), but since it's env-driven (`process.env.SECRET_COOKIE_PASSWORD`, not in-repo), confirm no environment (dev/test/staging/prod) has a secret under 32 chars.

---

## Validation Checks

### Per-workspace results (Build · Typecheck · Test)

| Workspace | Build | Typecheck | Tests |
|---|---|---|---|
| `@proofed/shared` | ✅ | ✅ | ⚠️ 1236/1237 (1 locale failure) |
| `@proofed/wysiwyg-editor` | ✅ (45s) | ✅ | ✅ 245 passed, 4 skipped |
| `@proofed/creative-portal` | ✅ (454s) | ✅ | ✅ 1660/1660 |
| `@proofed/customer-portal` | ✅ (294s) | ✅ | ✅ 335/335 |
| `@proofed/storybook` | ✅ (26s) | ✅ | — (no tests) |
| **TOTAL** | **4/4 ✅** | **5/5 ✅** | **3576 pass / 1 fail / 4 skip** |

The single test failure is `@proofed/shared` `formatWordQuantity.test.ts`, an `en_IN`-locale artifact (passes 9/9 under `en_US`/CI; file byte-identical to develop). `lint` remains ❌ only in `@proofed/wysiwyg-editor` (pre-existing prettier formatting, out of scope — see row below).

### Suite-level

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ❌ | 1 failure: `@proofed/shared` `formatWordQuantity.test.ts` expects `1,000,000` but got `10,00,000`. **Locale-driven** — the runner's locale is `en_IN`; the test passes 9/9 under `en_US` (CI/UTC), and the file is identical to develop. Not a code regression. creative-portal **1660/1660** ✅, customer-portal **335/335** ✅ |
| `npx turbo run typecheck` | ✅ | 5/5 workspaces, 0 errors |
| `npx turbo run lint` | ❌ | 63 `prettier/prettier` (formatting-only) errors in `@proofed/wysiwyg-editor`, across **5 files**: `components/molecules/AiChangeBox/index.tsx`, `components/molecules/CommentsContainer/utils.ts` + `formatIndividualDiffs.test.ts`, `contexts/EditorContext/hooks.ts`, `extensions/comments/index.ts`. **Pre-existing on develop** — all 5 files are byte-identical to develop **and** the lint toolchain is unchanged by the PR (`prettier ^3.2.5`, `eslint ^8.56.0`, `eslint-plugin-prettier ^5.1.3`), so develop produces the same 63 errors. The PR/merge did not touch them; auto-fixable via `--fix`. Out of PR scope. |
| `npx turbo run build` | ✅ | **4/4 workspaces pass** after the file-type fix (`e4460091f`). Both customer-portal and creative-portal build clean. (Was ❌ on `7cf739c1c` — Issue 1.) |

---

## Tests

- ✅ Typecheck green across all 5 workspaces under TS 6.0.2 — strong signal the config migration is correct.
- ✅ creative-portal (1660), customer-portal (335), and wysiwyg (245 + 4 skipped) unit suites pass; shared 1236/1237 (the 1 failure is the locale artifact below).
- ⚠️ The PR adds no new automated tests, but as a tooling/config upgrade the existing suites are the appropriate coverage. Acceptable for this ticket type.
- ⚠️ The single shared test failure is environment/locale, not code — but worth pinning `Intl` locale (`en-US`) in `formatWordQuantity` or its test setup so it is deterministic across dev machines (pre-existing on develop; track separately).
- ✅ **Build now passes (4/4)** after the file-type fix. The PR's own manual test items (Toast, Swiper carousel, file upload + mime detection, Google Picker, Sentry, **auth/login** for iron-session v8) remain unchecked and must be exercised — file upload/mime detection and auth especially.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Config/type migration is sound; build blocker fixed; iron-session v6→v8 migrated correctly; Sentry + Storybook reqs deferred to separate tickets by decision |
| Regression risk | ⚠️ Medium — swiper (3 majors), react-toastify (1 major), iron-session (2 majors, may force one-time re-login) need QA; bundler resolution can surface more raw-ESM packages |
| Tests | ⚠️ Suites pass except 1 locale-env failure; no new tests (acceptable for tooling) |
| Code quality | ✅ Clean, well-documented migrations |
| Validation suite | ✅ `build` 4/4 + `typecheck` 5/5; remaining `lint`/`test` failures are pre-existing on develop (out of scope) |
| Mergeable state | ✅ Builds clean; no open blockers (Sentry/Storybook deferred by decision; iron-session needs pre-deploy verification, not a merge blocker) |

---

## Recommendation

**Approve** — the build blocker is fixed (`e4460091f`, build now 4/4 green) and the two outstanding Jira requirements (Sentry, Storybook) are intentionally deferred to separate tickets per team decision. No open blockers remain.

1. ✅ **Done — Blocker (Issue 1):** `packages/shared` file-type aligned to `19.6.0`; `npx turbo run build` green for both portals.
2. ⏸️ **Deferred (Issue 2 — Sentry req #3):** `transpileClientSDK: false` deferred to a separate ticket. Housekeeping: correct the PR description, which claims `false` while the code is still `true`.
3. ⏸️ **Deferred (Issue 3 — Storybook req #2):** creative-portal alpha alignment deferred to a separate ticket. Update the Jira scope so req #2 is formally tracked rather than silently dropped.
4. **iron-session v6→v8 (Issue 5):** migration code is correct and type-checks, but (a) correct the PR description, which wrongly claims no v8 upgrade; (b) verify before deploy that existing sessions degrade gracefully (likely one-time forced re-login due to the v6→v8 cookie seal-format change) and that `SECRET_COOKIE_PASSWORD` is ≥ 32 chars in every environment.
5. **Before merge, run the PR's manual QA checklist** — Toast (all 4 types + maintenance banner), OrderJobs swiper carousel, customer-portal file upload + mime detection, Google Picker, Sentry reporting, **and an auth/login round-trip** (iron-session v8).
6. **Track separately (not blockers, pre-existing on develop):** the `wysiwyg` prettier lint errors and the `formatWordQuantity` locale-dependent test. Optional cleanup: drop the now-unused `file-type` declaration from `apps/customer-portal/package.json` (Issue 4).

_Note: this branch carries a fresh `develop` merge (`7cf739c1c`) that was resolved prior to review, plus the file-type build fix (`e4460091f`). The original build failure was independent of the merge — file-type `^22` predated it (commit `02be61c37`)._
