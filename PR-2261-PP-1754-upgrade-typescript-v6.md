# PR Review: PP-1754: Upgrade TypeScript to v6 and development tooling

**PR:** https://github.com/Proofed/B2BWebserver/pull/2261
**Jira:** https://proofed.atlassian.net/browse/PP-1754
**Status:** In Progress

> **Update (2026-04-20):** Sentry 7 → 10 upgrade has been reverted on this branch (commits `05c364a1a`, `7e5cd7c63`). `@sentry/nextjs` and `@sentry/types` are back at `7.73.0`. All Sentry-related review items below are retained for history but marked **[STALE — Sentry reverted]** where they no longer apply.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Upgrade TypeScript | 5.3.3 → 6.0.2 across all workspaces, no `ignoreDeprecations` escape hatch | ✅ Addressed |
| Upgrade Turborepo | 2.3.3 → 2.9.6 | ✅ Addressed |
| Upgrade Playwright | 1.45.1 → 1.59.1 | ✅ Addressed |
| Resolve `@types/node` mismatch | Set `types: ["node"]` in base tsconfig, explicit per-workspace `types` | ✅ Addressed |
| Sentry config: disable client SDK transpilation | **[STALE — Sentry reverted]** Sentry remains on v7; `transpileClientSDK: true` kept as the v7 default | ⚠️ N/A |
| Storybook alignment (remove alpha, align versions) | Not addressed in this PR | ❌ Missing |
| All packages must build and test successfully | PR description says typecheck/build/test all pass | ✅ Addressed |

**Scope beyond Jira ticket:**
- ~~Sentry 7 → 10 upgrade (major, with OpenTelemetry integration)~~ **[REVERTED]** — kept on v7
- axios 0.27 → 1
- date-fns 2 → 4 + date-fns-tz v3
- iron-session 6 → 8
- swiper 8 → 11 (3 major versions)
- react-toastify 9 → 10
- framer-motion → v12
- recharts 2 → 3
- husky 8 → 9
- stripe 14 → 22
- `moduleResolution: "node"` → `"bundler"` migration

> Many of these are justified — `moduleResolution: "bundler"` requires packages with proper `exports` fields, forcing swiper and react-toastify upgrades. However, the scope is significantly larger than the Jira ticket describes. This should be documented on the ticket.

---

## Architecture Analysis

This PR makes a foundational change: migrating `moduleResolution` from `"node"` to `"bundler"`. This is the modern default for bundled apps and respects `package.json` `exports` fields — which cascades into requiring several library upgrades (swiper, react-toastify) whose old versions lacked proper `exports.types` conditions.

The approach is sound:
- **tsconfig changes** follow the `ts5to6` migration tool's recommendations (`baseUrl` removal, explicit `paths`, explicit `types` arrays)
- **Library migrations** are consistent across all workspaces (date-fns-tz renames, iron-session API migration, Sentry v10 functional integration API)
- **Code fixes** are minimal and targeted (import path corrections, type assertion fixes for stricter inference)

The highest-risk changes are swiper 8→11 (visual/behavioral) and the `moduleResolution` switch (affects all import resolution). ~~Sentry 7→10~~ **[REVERTED]** — no longer a risk factor.

---

## Issues Found

### 1. `readFileSync` for MIME detection loads entire file into memory

**[File: packages/shared/utils/files/processWorkItemContentWithMetadata.ts]**
**Function/Class:** processWorkItemContentWithMetadata (streaming path, line 137)
**Severity:** medium
**Status:** ✅ Resolved — replaced `readFileSync` with `openSync`/`readSync` reading only first 4100 bytes
**Problem:** After streaming a file to disk via `Base64DecodeStream`, the code reads the entire file back into memory with `readFileSync(tempFilePath)` just to detect the MIME type. The previous `fileTypeFromFile` was streaming-based and only read the file header. For large files (100MB+ PDFs or ZIPs), this causes unnecessary memory pressure.
**Impact:** Potential OOM on large file uploads in the streaming code path. The base64 string path (line 147) doesn't have this issue since the buffer is already in memory.
**Fix:** Read only the first 4100 bytes — that's all `file-type` needs for magic number detection:

```typescript
import { openSync, readSync, closeSync } from "fs";

const fd = openSync(tempFilePath, "r");
const header = Buffer.alloc(4100);
readSync(fd, header, 0, 4100, 0);
closeSync(fd);
const ft = await fileTypeFromBuffer(header);
```

### 2. `@types/react-datepicker` version mismatch with react-datepicker v9

**[File: apps/customer-portal/package.json]**
**Function/Class:** N/A (package.json)
**Severity:** medium
**Status:** ✅ Resolved — removed `@types/react-datepicker` from customer-portal devDependencies
**Problem:** `@types/react-datepicker` is at `^6.0.0` but `react-datepicker` was bumped to `^9.1.0`. Starting from v7, react-datepicker bundles its own TypeScript types. The stale `@types` package may provide conflicting type definitions.
**Impact:** Could cause confusing type errors or mask API changes in react-datepicker v9.
**Fix:** Remove `@types/react-datepicker` from devDependencies — react-datepicker v9 ships its own types.

### 3. Empty string fallback for nullable Google Picker file name

**[File: apps/customer-portal/components/organisms/OrderCreation/index.tsx]**
**Function/Class:** OrderCreation component
**Severity:** low
**Status:** ✅ Resolved — changed fallback to `"Untitled"`
**Problem:** `(file.name ?? "").substring(...)` uses `""` as fallback when `file.name` is undefined (google.picker v0.0.52 made `name` nullable). This could create a document with an empty `title`.
**Impact:** If a Google Picker document has no `name`, the file would be created with an empty title.
**Fix:** Consider using a fallback like `"Untitled"` instead of `""`.

### 4. Fragile `as` casts in recharts tick props

**[File: apps/customer-portal/components/pages/reports/usage/partials/UsageChart/index.tsx]**
**Function/Class:** UsageChart
**Severity:** low
**Status:** ✅ Resolved — replaced casts with recharts' exported `YAxisTickContentProps` type; values explicitly coerced with `Number(...)` at the call boundary
**Problem:** Recharts v3 `tick` prop migration uses `as number` and `as { value: number }` casts that bypass type safety.
**Impact:** Low — recharts tick API is stable, but fragile if upgraded again.
**Fix:** Consider using recharts' exported `TickProps` type instead of casts.

### 5. Dead dependencies still present in customer-portal

**[File: apps/customer-portal/package.json]**
**Function/Class:** N/A (package.json)
**Severity:** low
**Status:** ⬜ Deferred to PP-1752 (as originally planned) — `core-js`, `node-fetch`, `@types/node-fetch`, `@mdx-js/react` still present; scope of PP-1754 not expanded
**Problem:** `core-js`, `node-fetch`, `@types/node-fetch` still listed despite `IMPROVEMENTS.md` flagging them as dead/phantom. `@mdx-js/react: "1"` also still in creative-portal.
**Impact:** Unnecessary bundle bloat.
**Fix:** Deferred to PP-1752 as planned — ensure that ticket tracks these explicitly.

### 6. Storybook alignment not addressed

**[File: N/A]**
**Function/Class:** N/A
**Severity:** low
**Status:** ⬜ Deferred to a separate Storybook ticket (as noted in PR description); Jira PP-1754 still needs explicit update
**Problem:** Jira ticket requirement #2 ("Remove alpha version of Storybook, align Storybook version across all portals") is not addressed.
**Impact:** Jira requirement gap.
**Fix:** PR description already notes "Storybook 8 — out of scope, separate tickets". Update the Jira ticket to reflect this deferral explicitly.

---

## Tests

- ✅ Test mocks updated for stricter TS 6.0 inference (`useGetAllOrderSortedJobs.test.ts`, `hooks.test.ts`)
- ✅ date-fns-tz mocks updated (`__mocks__/date-fns-tz.ts`)
- ✅ Sentry context tests updated for v10 API (`sentryContext.test.ts`, `sentryScrubber.test.ts`)
- ✅ All `toast.TYPE.*` enum references fully migrated to string literals
- ✅ All `react-dropzone/.` import paths fixed
- ✅ PR description reports: typecheck 0 errors, builds succeed, tests pass
- ⚠️ No new tests for iron-session v8 migration — existing coverage may suffice but manual verification needed
- ⚠️ Swiper 8→11 (3 major versions) requires manual UI testing — correctly flagged in PR test plan
- ⚠️ react-toastify 10 requires manual UI testing — correctly flagged in PR test plan

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ All code changes are mechanically correct migrations |
| Regression risk | ⚠️ Medium — swiper (3 major versions), Sentry (7→10), moduleResolution switch are high-risk areas requiring manual QA |
| Tests | ⚠️ Automated tests pass and were updated; highest-risk changes (swiper, toastify, iron-session) need manual verification |
| Code quality | ✅ Clean, consistent migration patterns across workspaces |
| Mergeable state | ✅ Clean |

---

## Package Changes (all 6 workspaces)

### 1. `package.json` (root)

| Package | Change |
|---|---|
| @playwright/test | 1.45.1 → 1.59.1 (also in resolutions) |
| concurrently | ^9.1.2 → ^9.2.1 |
| husky | ^8.0.0 → ^9.1.7 |
| lint-staged | ^15.2.8 → ^16.4.0 |
| turbo | 2.3.3 → 2.9.6 |

### 2. `apps/creative-portal/package.json`

**dependencies:**

| Package | Change |
|---|---|
| @azure/storage-blob | ^12.12.0 → ^12.31.0 |
| @emotion/react | ^11.11.4 → ^11.14.0 |
| @emotion/styled | ^11.3.0 → ^11.14.1 |
| @sentry/nextjs | 7.73.0 → ^10.48.0 |
| @stripe/stripe-js | ^2.2.2 → ^9.2.0 |
| @types/cookie | added |
| @types/papaparse | ^5.3.5 → ^5.5.2 |
| autoprefixer | ^10.4.8 → ^10.5.0 |
| axios | ^0.27.2 → ^1.15.0 |
| caniuse-lite | ^1.0.30001680 → ^1.0.30001788 |
| clsx | ^1.2.1 → ^2.1.1 |
| cookie | ^0.6.0 → ^1.1.1 |
| date-fns | ^2.29.2 → ^4.1.0 |
| date-fns-tz | ^2.0.0 → ^3.2.0 |
| dotenv | ^16.5.0 → ^17.4.2 |
| formidable | ^2.0.1 → ^3.5.4 |
| formik | ^2.2.9 → ^2.4.9 |
| iron-session | ^6.1.3 → ^8.0.4 |
| js-base64 | ^3.7.7 → ^3.7.8 |
| jsonwebtoken | ^9.0.2 → ^9.0.3 |
| lodash | ^4.17.21 → ^4.18.1 |
| node-html-parser | ^7.0.1 → ^7.1.0 |
| papaparse | ^5.3.2 → ^5.5.3 |
| postcss | ^8.4.16 → ^8.5.10 |
| react-avatar-editor | ^13.0.0 → ^15.1.0 |
| react-imask | ^7.1.3 → ^7.6.1 |
| react-paginate | ^8.1.3 → ^8.3.0 |
| react-phone-number-input | ^3.2.12 → ^3.4.16 |
| react-scroll | ^1.8.7 → ^1.9.3 |
| react-select | ^5.4.0 → ^5.10.2 |
| stripe | ^14.9.0 → ^22.0.1 |
| swiper | ^8.4.6 → ^11.2.6 |

**devDependencies:**

| Package | Change |
|---|---|
| @emotion/babel-plugin | ^11.10.2 → ^11.13.5 |
| @sentry/types | 7.73.0 → removed |
| @svgr/webpack | ^6.3.1 → ^8.1.0 |
| @types/formidable | ^2.0.5 → ^3.5.1 |
| @types/jsonwebtoken | ^9.0.6 → ^9.0.10 |
| @types/lodash.throttle | ^4.1.7 → ^4.1.9 |
| @types/node | 18.7.6 → ^20 |
| @types/react-avatar-editor | ^13.0.0 → ^13.0.4 |
| @types/react-phone-number-input | ^3.0.14 → ^3.1.37 |
| @types/react-scroll | ^1.8.4 → ^1.8.10 |
| @types/react-table | ^7.7.12 → ^7.7.20 |
| @types/react-text-mask | ^5.4.11 → ^5.4.14 |
| cross-env | ^7.0.3 → ^10.1.0 |
| typescript | 5.3.3 → 6.0.2 |

### 3. `apps/customer-portal/package.json`

**dependencies:**

| Package | Change |
|---|---|
| @sentry/nextjs | 7.73.0 → ^10.48.0 |
| file-type | 19.6.0 → ^22.0.1 |
| googleapis | ^136.0.0 → ^171.4.0 |
| mobx | ^6.13.6 → ^6.15.0 |
| react-datepicker | ^6.0.0 → ^9.1.0 |
| recharts | ^2.12.0 → ^3.8.1 |
| redoc | ^2.4.0 → ^2.5.2 |
| stream-json | ^1.9.1 → ^2.1.0 |
| styled-components | ^6.1.15 → ^6.4.0 |

**devDependencies:**

| Package | Change |
|---|---|
| @emotion/babel-plugin | ^11.10.2 → ^11.13.5 |
| @sentry/types | 7.73.0 → removed |
| @types/google.picker | ^0.0.42 → ^0.0.52 |
| typescript | 5.3.3 → 6.0.2 |

### 4. `apps/storybook/package.json`

| Package | Change |
|---|---|
| @emotion/react | ^11.11.4 → ^11.14.0 |
| typescript | 6.0.2 → added |

### 5. `packages/shared/package.json`

**dependencies:**

| Package | Change |
|---|---|
| @emotion/babel-plugin | ^11.10.2 → ^11.13.5 |
| @emotion/react | ^11.11.4 → ^11.14.0 |
| @emotion/styled | ^11.3.0 → ^11.14.1 |
| @types/formidable | ^3.4.5 → ^3.5.1 |
| dompurify | ^3.2.5 → ^3.4.0 |
| fast-xml-parser | ^5.0.8 → ^5.6.0 |
| formidable | ^3.5.2 → ^3.5.4 |
| framer-motion | 11.0.12 → ^12.38.0 |
| lodash | ^4.17.21 → ^4.18.1 |
| muhammara | ^5.3.0 → ^6.0.4 |
| otpauth | ^9.4.1 → ^9.5.0 |
| react-dropzone | ^14.2.2 → ^15.0.0 |
| react-hotjar | 6.2.0 → 6.3.1 |
| react-toastify | ^9.0.8 → ^10.0.6 |
| transliteration | ^2.6.0 → ^2.6.1 |
| usehooks-ts | ^3.1.0 → ^3.1.1 |

**devDependencies:**

| Package | Change |
|---|---|
| typescript | 6.0.2 → added |

> Note: `"exports": { "./*": "./*" }` field was removed.

### 6. `packages/wysiwyg/package.json`

**dependencies:**

| Package | Change |
|---|---|
| dompurify | ^3.2.5 → ^3.4.0 |
| js-base64 | ^3.7.7 → ^3.7.8 |
| react-toastify | 9.0.8 → 10.0.6 |
| y-websocket | ^2.1.0 → ^3.0.0 |
| yjs | ^13.6.24 → ^13.6.30 |

**devDependencies:**

| Package | Change |
|---|---|
| @rollup/plugin-commonjs | ^25.0.7 → ^29.0.2 |
| @rollup/plugin-node-resolve | ^15.2.3 → ^16.0.3 |
| @rollup/plugin-typescript | ^11.1.6 → ^12.3.0 |
| autoprefixer | ^10.4.17 → ^10.5.0 |
| clsx | ^2.1.0 → ^2.1.1 |
| postcss | ^8.4.35 → ^8.5.10 |
| postcss-import | ^16.1.1 → added |
| rollup | ^4.9.6 → ^4.60.1 |
| rollup-plugin-dts | ^6.1.0 → ^6.4.1 |
| tslib | ^2.6.2 → ^2.8.1 |
| typescript | ^5.3.3 → ^6.0.2 |

**Total: ~85 package changes across 6 workspaces.**

---

## Side-Effects Analysis

Deep analysis of every major version bump for runtime side effects, missed migration spots, and dependency declaration issues.

### Issues Found (action needed)

#### HIGH — Must fix before merge

| # | Package | Issue | File(s) | Status |
|---|---|---|---|---|
| 1 | **iron-session 6→8** | `iron-session` is only declared in `creative-portal/package.json`, but it's imported directly in `packages/shared` (session.ts, withApiMiddleware, login, MFA, etc.) and `customer-portal`. Relies on Yarn hoisting — fragile. | `packages/shared/package.json`, `apps/customer-portal/package.json` | ✅ Resolved — added `iron-session: ^8.0.4` to both `packages/shared/package.json` and `apps/customer-portal/package.json` |
| 2 | ~~**Sentry 7→10**~~ **[STALE — Sentry reverted]** | `wrapApiHandlerWithSentry` exists in v7 — no longer a concern since Sentry stayed on 7.73.0. | `packages/shared/api/utils/middlewares/withApiMiddleware/withApiMiddleware.ts` | ⚠️ N/A (stale) |
| 3 | **react-dropzone** | Version mismatch — creative-portal declares `^14.2.2`, shared declares `^15.0.0`. Yarn hoisting may mask this but it's fragile. | `apps/creative-portal/package.json` vs `packages/shared/package.json` | ✅ Resolved — bumped creative-portal `react-dropzone` to `^15.0.0` |

#### MEDIUM — Should fix

| # | Package | Issue | File(s) | Status |
|---|---|---|---|---|
| 4 | ~~**Sentry 7→10**~~ **[STALE — Sentry reverted]** | `transpileClientSDK` / `autoInstrumentServerFunctions` are valid v7 options — no action needed. | `packages/shared/scripts/nextConfig.js` | ⚠️ N/A (stale) |
| 5 | ~~**Sentry 7→10**~~ **[STALE — Sentry reverted]** | `getContext()` is the correct v7 mock API — no action needed. | `packages/shared/__mocks__/@sentry/nextjs.ts` | ⚠️ N/A (stale) |
| 6 | **axios 0.27→1** | `AxiosRequestHeaders` cast on `{orgId?: number}` in `services/customers/index.tsx`. The type changed in v1 — this cast is dubious. | `apps/creative-portal/services/customers/index.tsx` | ✅ Resolved — removed `AxiosRequestHeaders` cast; stringify `orgId`/`orgGroupId` before passing as headers |
| 7 | **@types/react-datepicker** | Still at `^6.0.0` but react-datepicker v9 bundles its own types. Stale `@types` may conflict. | `apps/customer-portal/package.json` | ✅ Resolved — removed `@types/react-datepicker` from customer-portal devDependencies |
| 8 | **file-type 19→22** | `readFileSync(entireFile)` for MIME detection — should read only first 4KB to avoid OOM on large uploads. | `packages/shared/utils/files/processWorkItemContentWithMetadata.ts` | ✅ Resolved — replaced with `openSync`+`readSync` reading only first 4100 bytes |

#### LOW — Nice to fix

| # | Package | Issue | File(s) | Status |
|---|---|---|---|---|
| 9 | **stream-json 1→2** | Zero source imports — unused dependency. | `apps/customer-portal/package.json` | ✅ Resolved — removed from customer-portal dependencies |
| 10 | **file-type** | Declared in customer-portal but no imports found there — unused. | `apps/customer-portal/package.json` | ✅ Resolved — removed from customer-portal and moved to `packages/shared` (where it is actually imported) |
| 11 | **muhammara 5→6** | Native binary rebuild — test PDF metadata injection in dev. | N/A | ⬜ Manual QA pending (runtime-only, no code change needed) |

### Clean — No side effects found

| Package | Verdict |
|---|---|
| **swiper 8→11** | All props valid, `Mousewheel` import path updated, CSS paths correct |
| **react-toastify 9→10** | All enum references migrated to string literals, 0 remaining |
| **framer-motion 11→12** | All transition types valid (`"spring"`, `"tween"`, `ease: "easeInOut"/"linear"`) |
| **date-fns 2→4 + date-fns-tz 2→3** | All 20+ files migrated (named imports, `toZonedTime`/`fromZonedTime`), 0 old names remaining |
| **recharts 2→3** | All chart components compatible, tick prop updated |
| **formidable 2→3** | All file access patterns already handle v3 array semantics |
| **stripe 14→22 + @stripe/stripe-js 2→9** | Identity API stable, `loadStripe`/`verifyIdentity` unchanged |
| **cookie 0.6→1.1.1** | Cookie names are simple `trustBrowser-{userId}` — passes v1 validation |
| **react-avatar-editor 13→15** | Already using v15 `AvatarEditorRef` type |
| **googleapis 136→171** | Stable Google API methods (Drive v3, Docs, Sheets, Slides) |
| **iron-session code migration** | All 23+ files correctly migrated from HOC wrappers to `getIronSession()` |
| **axios code migration** | `onUploadProgress.total` optional, numeric status checks, no `CancelToken` usage |
| **dotenv, cross-env, husky, lint-staged** | Config/dev-only, no runtime impact |
| **react-datepicker 6→9** | Only 1 usage site, all props valid (`selected`, `onChange`, `selectsRange`, `inline`, etc.) |
| **y-websocket 2→3** | Provider API unchanged |
| **Rollup plugins** | Plugin APIs backward-compatible for current config |

### Recommended priority order

1. ✅ **Add `iron-session` to `packages/shared/package.json` and `apps/customer-portal/package.json`** — **Done**
2. ✅ **Align react-dropzone** — bumped creative-portal to `^15.0.0` to match shared — **Done**
3. ✅ **Remove `@types/react-datepicker`** from customer-portal — **Done**
4. ✅ **Fix `readFileSync` for MIME detection** — now reads only first 4100 bytes — **Done**
5. ✅ **Remove unused deps** (`stream-json`, `file-type` in customer-portal) — **Done** (file-type moved to `packages/shared` where it is actually used)
6. ~~Verify `wrapApiHandlerWithSentry` exists in v10~~ **[STALE — Sentry reverted to v7]**
7. ~~Clean up Sentry mock and deprecated `nextConfig.js` options~~ **[STALE — Sentry reverted to v7]**

All non-stale Side-Effects items (1, 3, 6, 7, 8, 9, 10) are resolved. Item 11 (`muhammara 5→6` native-binary QA) is a runtime test that needs manual verification before merge.

---

## Recommendation

**Approve with suggestions**

1. ✅ **Add `iron-session` to `packages/shared/package.json` and `apps/customer-portal/package.json`** — **Done** (previously relied on Yarn hoisting from creative-portal).
2. ✅ **Align react-dropzone versions** — creative-portal bumped from `^14.2.2` to `^15.0.0` to match shared. **Done**
3. ✅ **Fix `readFileSync` for MIME detection** — now reads only the first 4100 bytes instead of the entire file. **Done**
4. ✅ **Remove `@types/react-datepicker`** from customer-portal — react-datepicker v9 ships its own types. **Done**
5. ✅ **Remove unused deps** — `stream-json` removed; `file-type` relocated from customer-portal to `packages/shared` (where it is actually imported). **Done**
6. ✅ **Remove fragile `AxiosRequestHeaders` cast in `services/customers/index.tsx`** — headers now explicitly stringified. **Done**
7. ✅ **Replace recharts tick `as` casts** with `YAxisTickContentProps` in `UsageChart`. **Done**
8. ✅ **Use `"Untitled"` fallback** for nullable Google Picker `file.name`. **Done**
9. ⬜ **Manual QA the test plan items** before merging — especially swiper carousel, toast notifications, iron-session login/logout flows, and `muhammara` PDF metadata injection.
10. ⬜ **Document scope expansion on the Jira ticket** — the PR goes well beyond the ticket's stated scope (justified, but should be documented for traceability).
11. ⬜ **Note Storybook deferral** — explicitly mark Jira requirement #2 as deferred to a separate ticket.

> ~~Verify `wrapApiHandlerWithSentry` in Sentry v10~~ and ~~Clean up Sentry config (deprecated options + stale mock)~~ — **[STALE]** Sentry 7→10 upgrade was reverted (commits `05c364a1a`, `7e5cd7c63`); these items no longer apply.

### Verification (on current working tree)
- `npx turbo run typecheck` → 5/5 workspaces pass (0 errors)
- ESLint on each file touched by this review — clean (pre-existing lint errors in `packages/wysiwyg` and `packages/shared` are unchanged by these edits)
- `npx turbo run test` → all suites green (950 shared, 706 creative-portal, plus customer-portal and wysiwyg)
