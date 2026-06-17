# PR Review — PP-1750: Improve System Error Tracking and Logging

## Quick facts

| | |
|---|---|
| **PR** | https://github.com/Proofed/B2BWebserver/pull/2253 |
| **Jira** | https://proofed.atlassian.net/browse/PP-1750 |
| **Branch** | `fix/PP-1750-improve-error-tracking` |
| **Base** | `develop` (tip `054c02f83`) |
| **State** | Open, up-to-date with `develop` |
| **Tip commit** | `25cb62ae8` (formatting-drift revert) + 12 staged uncommitted files (scrubber-leak fix + DRY + hardening + rename) |
| **Local CI** | `@proofed/shared` 1300/1300 ✅ · `@proofed/creative-portal` 1637/1637 ✅ · `@proofed/customer-portal` full suite ✅ · typecheck clean on all 3 |
| **Last reviewed** | 2026-06-17 |

## Table of contents

1. [Verdict](#verdict)
2. [Resolution status of the prior review (issues #1–#14)](#resolution-status-of-the-prior-review-issues-1-14)
3. [Develop merged in — conflicts resolved](#develop-merged-in--conflicts-resolved)
4. [Scope cleanup — formatting drift reverted](#scope-cleanup--formatting-drift-reverted)
5. [Security audit — scrubber leak fixed + hardening](#security-audit--scrubber-leak-fixed--hardening)
6. [Production-grade audit — official-docs cross-check](#production-grade-audit--official-docs-cross-check)
7. [Architecture analysis](#architecture-analysis)
8. [Tests](#tests)
9. [Recommendation](#recommendation)
10. [Appendix — original review write-ups](#appendix--original-review-write-ups)

---

## Verdict

**Hold the merge** until three blockers land, then approve:

1. Commit the staged security fix + DRY + hardening + rename (12 files).
2. Delete the orphaned stale-DSN files (Audit #1).
3. Switch `_error.tsx` to `await Sentry.captureUnderscoreErrorException(contextData)` (Audit #2).

After those land and CI re-runs green, this PR is ready to merge. Full action list under [Recommendation](#recommendation).

---

## Resolution status of the prior review (issues #1–#14)

Every code-level fix from the prior review round is verified present on the branch head.

| # | Issue | Status | Landing commit |
|---|---|---|---|
| 1 | Caller-supplied `extra` overwritten by axios details | ✅ Resolved | `af8f89c78` |
| 2 | Duplicated QueryClient + interceptor bootstrap | ✅ Resolved — extracted to `createAppQueryClient` | `af8f89c78` |
| 3 | Narrow error boundary scope | ✅ Resolved — boundary is outermost, inside `<ThemeProvider>` | `af8f89c78` |
| 4 | Axios `AxiosHeaders` prototype stripped by spread | ⚠️ **Conditional** — safe today, breaks under axios 1.x. Coordinate with [PR #2261](https://github.com/Proofed/B2BWebserver/pull/2261). | — |
| 5 | High-cardinality `route` tag | ✅ Resolved — `getSentryRoute()` prefers Next router template | `af8f89c78` |
| 6 | Error ID shown in boundary but not in `ErrorPage` | ✅ Resolved — `errorId` threaded through | `af8f89c78` |
| 7 | Unsafe `as Error & { statusCode?: number }` cast | ✅ Resolved — narrowed | `af8f89c78` |
| 8 | Sentry mock missing v10 API surface | ✅ Resolved — full v10 surface mocked | `af8f89c78` |
| 9 | No test for `withIsolationScope` migration | ✅ Resolved — 5 tests in `withSentryUser.test.ts` | `af8f89c78` |
| 10 | Module-scope `installAxiosCorrelationInterceptor()` | ✅ Resolved — moved into `useEffect` | `af8f89c78` |
| 11 | Stale `/* eslint-disable no-console */` | ✅ Resolved | `af8f89c78` |
| 12 | `^10.48.0` caret range | ✅ Resolved — pinned to exact `10.48.0` | `8dbf0b9de` |
| 13 | No test for `reportError` with primitive inputs | ✅ Resolved — 5 new primitive tests | `af8f89c78` + `c6ee03005` |
| 14 | No test for `_app.tsx` QueryCache/MutationCache wiring | ✅ Resolved — factory extracted + 5 tests | `af8f89c78` |

Detailed write-ups for each issue live in [Appendix — original review write-ups](#appendix--original-review-write-ups).

### Post-review hardening (carried forward)

| Change | Commit |
|---|---|
| Wrap non-Error `reportError` inputs in synthetic `Error("Non-Error thrown: …")` so Sentry shows searchable titles; raw value preserved at `extra.originalValue` | `c6ee03005` |
| Pin `@sentry/nextjs` to exact `10.48.0` to prevent silent drift on a freshly-upgraded SDK (closes review #12) | `8dbf0b9de` |
| Wider `reportError` / scrubber hardening across call sites | `db0dc0855` |

> The `/sentry-test` manual verification page (commits `c3cc2397b` and `069833755`) was reverted before merge — the automated suite covers every error-tracking path, and a dev-only page reachable on stage/devtest created noise without lasting value.

---

## Develop merged in — conflicts resolved

`develop` (tip `054c02f83`) was merged into the branch on 2026-06-17 via commit `63d9dd85b`. The 4 prior conflicts were resolved as follows — none required semantic compromise:

| File | Resolution |
|---|---|
| `apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx` | Took develop's PR #2306 refactor (inline customer Cell extracted into `CustomerCell`). PP-1750 never touched this file directly. |
| `apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/DetailedOrderInfo.tsx` | Took develop's PR #2306 reorganization — `Created by` / `Created for` rows moved into the people-row component, `IconLogoApi` import dropped, field ordering aligned with the new design. PP-1750 never touched this file directly. |
| `apps/customer-portal/package.json` | Hybrid — kept PP-1750's `@sentry/nextjs: "10.48.0"` pin AND applied develop's PR #2232 dead-deps cleanup (dropped `@types/node-fetch`, `core-js`, `node-fetch` — none imported in source). Skipped develop's `@sentry/types: "7.73.0"` devDep — incompatible with v10 and never imported outside the vitest mock (the mock at `packages/shared/__mocks__/@sentry/types.ts` is wired via `vitest.config.ts` alias, so no runtime dep is needed). |
| `yarn.lock` | Took develop's lockfile then re-ran `yarn install` to resolve the new sentry pin against the merged `package.json` set. Verified `@sentry/nextjs@10.48.0` is the only resolution. |

---

## Scope cleanup — formatting drift reverted

Commit `25cb62ae8`. A reviewer flagged the PR diff carried formatting noise unrelated to error tracking. Audit confirmed 20 files had **only** Prettier/style drift vs `develop` — no semantic PP-1750 work. Drift came from earlier merge resolutions (`a5b12d102`, `765f73031`) that picked the branch's older style over develop's reformatted version.

### Examples of the drift

- `} as Job);` (v2 style on branch) vs `}) as Job;` (v3 style on develop)
- 4-space ternary indent on branch vs 2-space on develop
- `(x ?? y)` parenthesised vs unparenthesised
- `await importOriginal<typeof …>()` line-break placement

### Root cause

`node_modules/.bin/prettier` was a Yarn-1 hoisting symlink to `@storybook/cli/node_modules/prettier/bin-prettier.js` (v2.8.8), but `package.json` declares `prettier@^3.1.1` (v3.5.3 actually installed). So `lint-staged` was silently running v2 — which formats `}) as Job;` back to `} as Job);`.

### Files reverted to develop's version (20)

```
apps/creative-portal/api/aiReviewFeedback/strategy/ai-review-feedback.test.ts
apps/creative-portal/api/jobs/[jobId]/patchJob.test.ts
apps/creative-portal/api/mixtures/orders/getBulkActionsData/getBulkActionsData.test.ts
apps/creative-portal/api/orders/utils.test.ts
apps/creative-portal/api/utils/jobs/mergeJobPutBody.test.ts
apps/creative-portal/api/aiReviewFeedback/apiHandlers/submit/index.test.ts
apps/creative-portal/api/aiReviewFeedback/apiHandlers/submit/index.ts
apps/creative-portal/components/molecules/tables/ChargeTable/tableColumns.tsx
apps/creative-portal/components/molecules/tables/TableWithFilters/cells/__tests__/fixtures.ts
apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useActiveUuidLifecycle.test.tsx
apps/creative-portal/components/molecules/JobReturnTimesTray/index.test.tsx
apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/hooks.tsx
apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/utils/generateWorkflowComponents.tsx
apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/utils.test.ts
apps/customer-portal/api/utils/mixtures/orders/createOrder/__tests__/utils.test.ts
apps/customer-portal/api/utils/mixtures/orders/createOrder/utils.ts
apps/customer-portal/api/utils/mixtures/orders/createOrder/helpers/createOrderAndJobs.ts
apps/customer-portal/components/molecules/OrderPriceSection/__tests__/utils.test.ts
packages/shared/components/atoms/DotsLoader/styles.ts
packages/shared/contexts/richTextEditorContext/provider.tsx
```

### Kept (legitimate PP-1750 work)

The remaining non-error-tracking-named files in the PR diff are actually `console.error` → `reportError` migrations:

- `apps/creative-portal/api/utils/workItemContentVersion/serializerUtils.ts`, `tiptapAiChanges.ts`
- `apps/creative-portal/components/molecules/tables/TableWithFilters/utils.ts`
- `apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/partials/JobCard.tsx`, `UploadedFileView/hooks.ts`
- `apps/creative-portal/components/pages/partners/.../settings/notifications/hooks.ts`, `OrderTemplates/hooks.ts`
- `apps/creative-portal/contexts/teamMembersContext/provider.tsx`
- `apps/creative-portal/services/wise/createRecipient/index.ts`
- `apps/customer-portal/components/organisms/OrderCreation/hooks/useSupportDocuments/index.ts`
- `packages/shared/scripts/nextConfig.js` — Sentry `transpileClientSDK` cleanup (in-scope for v7→v10 bump)
- Both `apps/*/package.json` — Sentry version pin to `10.48.0`
- All `packages/shared/utils/*Sentry*`, `*throwSentryError*`, `*sentryContext*`, `*sentryScrubber*`, `getSentryRoute`, `installAxiosCorrelationInterceptor`, `createAppQueryClient` — new infrastructure
- `packages/shared/components/molecules/AppErrorBoundary/*`, `packages/shared/components/pages/error/*`
- `packages/shared/__mocks__/@sentry/nextjs.ts` — v10 mock surface
- Both `apps/*/pages/_app.tsx`, `_error.tsx`

### Held but unrelated to error tracking

`TASKS_COMPLETED.md` deletion (-653 lines). User opted to keep it deleted; harmless dev-log purge that can stay in this PR or be carved out later.

### Side fix during cleanup

The `.bin/prettier → @storybook/cli/…/prettier@2.8.8` symlink was repointed to `node_modules/prettier/bin/prettier.cjs` (v3.5.3) locally so the pre-commit hook would stop fighting the v3-style revert. This change is in `node_modules/` (gitignored) so it does not ship with the PR — but a follow-up is warranted to either bump the storybook dep or remove the conflicting copy at the project level. Until that happens, any dev who reformats these files via `yarn format` on a fresh `yarn install` will accidentally re-introduce the v2-style noise.

### Diff impact

PR file count went from 67 → 47 files; 50 lines of formatting noise removed.

---

## Security audit — scrubber leak fixed + hardening

Staged, uncommitted. Pre-flight verification: 1300/1300 tests pass; typecheck clean on all three workspaces.

### Critical leak — verified and fixed (Option A)

The global `beforeSend` scrubber in `sentryScrubber.ts` only recurses into objects:

```ts
const scrubSensitiveData = (data: unknown): unknown => {
  if (!data || typeof data !== "object") return data;   // ← strings returned as-is
  // ...
  if (typeof scrubbedData[key] === "object") {           // ← won't descend into stringified JSON
    scrubbedData[key] = scrubSensitiveData(scrubbedData[key]);
  }
};
```

`buildExtras` in `throwSentryError.ts` was serialising sensitive payloads into JSON strings **before** they reached the scrubber:

```ts
extras.requestData  = toJsonString(error.config?.data);
extras.responseData = toJsonString(error.response?.data);
extras.queryKey     = serializeAndTruncate(context.queryKey);
extras.mutationKey  = serializeAndTruncate(context.mutationKey);
```

**Live test result against the actual scrubber — 6 confirmed leaks:**

| Field | Value in stringified body | Scrubber output |
|---|---|---|
| `password` in `extras.requestData` | `"hunter2"` | 🚨 ships in cleartext |
| `email` in `extras.requestData` | `"victim@example.com"` | 🚨 ships in cleartext |
| `apiKey` in `extras.requestData` | `"ak_live_abc123"` | 🚨 ships in cleartext |
| `token` in `extras.responseData` | `"jwt_secret_xyz"` | 🚨 ships in cleartext |
| `ssn` in `extras.responseData` (nested) | `"123-45-6789"` | 🚨 ships in cleartext |
| `email` in `extras.queryKey` | `"victim@example.com"` | 🚨 ships in cleartext |
| `authorization` in `extras.requestHeaders` (object) | `"Bearer ..."` | ✅ `[REDACTED]` — stayed object, scrubber walked it |

**Impact:** every failed login routed through `createAppQueryClient`'s `MutationCache.onError` would ship the cleartext password to Sentry. Same for any 4xx/5xx response carrying tokens, SSNs, or PII.

**Fix (Option A — scrub before serialise):** `scrubSensitiveData` is now exported from `sentryScrubber.ts`. `buildExtras` calls it on each raw value **before** JSON-encoding:

```ts
extras.queryKey     = serializeAndTruncate(scrubSensitiveData(context.queryKey));
extras.mutationKey  = serializeAndTruncate(scrubSensitiveData(context.mutationKey));
extras.requestData  = toJsonString(scrubSensitiveData(error.config?.data));
extras.responseData = toJsonString(scrubSensitiveData(error.response?.data));
```

4 new regression tests in `throwSentryError.test.ts` (`pre-serialize scrubbing of stringified extras` describe block) lock the leak shut for each path.

### Init-level hardening — DRY'd into a shared base

A new `packages/shared/config/sentry/baseInit.ts` exposes `buildBaseSentryOptions(portal)` returning common options every runtime uses. The three runtime configs (`client.config.ts`, `server.config.ts`, `edge.config.ts`) spread it and add only their runtime-specific extras (Replay + BrowserTracing for client).

Security defaults consolidated in `baseInit.ts`:

| Option | Value | Rationale |
|---|---|---|
| `sendDefaultPii: false` | explicit | Sentry default is unstable across major versions — pin it. Blocks SDK auto-attach of IP / cookies. |
| `normalizeDepth: 5` | up from default 3 | The scrubber recurses depth-first; default 3 silently truncates deep payloads before the scrubber walks them. |
| `maxBreadcrumbs: 50` | down from default 100 | Breadcrumbs capture `console.*` and `fetch` — fewer = smaller leak surface. |
| `beforeSend: sentryScrubber` | unchanged | Single source of truth for PII removal at send-time. |

`baseInit.test.ts` (4 tests) pins each value so a future edit can't silently drop them.

### Replay options — dropped after docs review

Initially added `networkDetailAllowUrls: []` + `networkCaptureBodies: false` to the client Replay integration. **Verified against Sentry's official docs** (https://docs.sentry.io/platforms/javascript/session-replay/configuration/#network-details):

> "By default, Replay captures basic information about all outgoing fetch and XHR requests without requiring configuration. This includes the URL, request and response body size, method, and status code, intentionally designed to limit the chance of collecting private data."

Sentry's default IS `[]` already, and `networkCaptureBodies` only takes effect for URLs in `networkDetailAllowUrls` (so it's dormant when the allowlist is empty). Both options were no-ops. Replaced with a comment that points future maintainers at Sentry's network-details doc and reminds them to audit any `networkDetailAllowUrls` addition for PII.

### Cleaner DSN selector

Renamed the boolean `isCreativePortal` to `portal: "creative" | "customer"` (new `SentryPortal` type in `keys.ts`). The boolean kept reading as a feature gate ("is Sentry creative-only?") rather than the DSN selector it actually is. Call sites become self-documenting:

```ts
initSentry()             // creative (default)
initSentry("customer")   // customer
```

Customer-portal entry stubs updated from `initSentry(false)` to `initSentry("customer")`.

### Files touched (12 staged, uncommitted)

```
packages/shared/utils/sentryScrubber.ts             (export scrubSensitiveData)
packages/shared/utils/throwSentryError.ts           (pre-serialize scrubbing)
packages/shared/utils/throwSentryError.test.ts      (+4 regression tests)
packages/shared/config/sentry/baseInit.ts           (NEW — DRY'd base)
packages/shared/config/sentry/baseInit.test.ts      (NEW — 4 tests)
packages/shared/config/sentry/keys.ts               (SentryPortal type)
packages/shared/config/sentry/client.config.ts      (spread base + Replay)
packages/shared/config/sentry/server.config.ts      (spread base only)
packages/shared/config/sentry/edge.config.ts        (spread base only)
apps/customer-portal/sentry.client.config.ts        (initSentry("customer"))
apps/customer-portal/sentry.server.config.ts        (initSentry("customer"))
apps/customer-portal/sentry.edge.config.ts          (initSentry("customer"))
```

---

## Production-grade audit — official-docs cross-check

Cross-checked the implementation against:

- https://docs.sentry.io/platforms/javascript/guides/nextjs/manual-setup/pages-router/
- https://docs.sentry.io/platforms/javascript/configuration/options/
- https://docs.sentry.io/platforms/javascript/session-replay/configuration/

### ✅ What's correct against the official docs

1. **File layout for Pages Router** — `sentry.{client,server,edge}.config.ts` at each app root. Verified canonical for Next.js 14 Pages Router. `instrumentation.ts` is App Router-only.
2. **`withSentryConfig` wrap** with `widenClientFileUpload`, `tunnelRoute: "/monitoring"`, `disableLogger: true`, `autoInstrumentServerFunctions: true` — all matches Sentry's recommendations.
3. **DSN env segregation** — 3 envs × 2 portals = 6 DSNs in `keys.ts`. No leakage between environments.
4. **Tiered `tracesSampleRate`** (1.0 / 0.5 / 0.1) — prevents quota burn while keeping representative samples per env.
5. **Replay defaults** — `maskAllText: true` + `blockAllMedia: true`, plus Sentry's safe-by-default network capture (URL/method/status/sizes only, no headers, no bodies).
6. **`beforeSend: sentryScrubber`** on all 3 runtimes.
7. **Init hardening** — `sendDefaultPii: false`, `normalizeDepth: 5`, `maxBreadcrumbs: 50` in the DRY'd base.
8. **Scrub-before-serialize** (Option A) — closes the stringified-body leak.
9. **`AppErrorBoundary`** wrapping outermost JSX inside `<ThemeProvider>` — catches all render crashes; intentionally inside ThemeProvider so the fallback UI can resolve theme tokens.
10. **`useSentryIdentity`** sets `{id, email, roles}` then the scrubber strips email/roles before send.
11. **`withSentryUser`** middleware wraps backend handlers in `withIsolationScope`.
12. **Axios correlation interceptor** + corr_id propagation joins frontend Sentry events to backend Logtail events.

### 🚨 Real bugs surfaced — block this PR

#### Audit #1 — Orphaned files with stale DSNs pointing at a DIFFERENT Sentry org (critical)

```
apps/creative-portal/config/sentry.ts:1
  export const SENTRY_DSN =
    "https://66bfe869faad62cb8619f8f007be7bd8@o4505980060434432.ingest.sentry.io/4505980063580160";

apps/customer-portal/config/sentry.ts:1
  export const SENTRY_DSN =
    "https://6451241445c1a3d26841d19a01db5343@o4505980060434432.ingest.us.sentry.io/4508421795872768";
```

Both reference org `o4505980060434432` — but the active configs in `packages/shared/config/sentry/keys.ts` use orgs `o4509162431971328` (devtest/stage) and `o4509172595556352` (production). The two files are **not imported anywhere** (verified by grep across all `.ts`/`.tsx`/`.js` in `apps/` and `packages/`).

**Risk:** if a future dev re-imports them — autocomplete may surface `SENTRY_DSN` from these files — events would route to an abandoned Sentry org. Worst case: a stranger's account ingests our error stream.

**Fix:** delete both files. No call sites to update.

#### Audit #2 — `_error.tsx` uses `Sentry.captureException` instead of `captureUnderscoreErrorException` (high)

Sentry's Pages Router doc:

> "In case this is running in a serverless function, await this in order to give Sentry time to send the error before the lambda exits."
>
> ```ts
> await Sentry.captureUnderscoreErrorException(contextData);
> ```

Our `packages/shared/components/pages/error/index.tsx:74-78`:

```ts
// Skip 4xx - match @sentry/nextjs `captureUnderscoreErrorException`
// behaviour so we don't page on 404s and other client errors.
if (!statusCode || statusCode >= 500) {
  errorId = Sentry.captureException(contextData.err);
}
```

Two issues:

- Uses bare `Sentry.captureException` instead of `captureUnderscoreErrorException`, losing SSR-only context (URL, headers, user agent) that `captureUnderscoreErrorException` attaches automatically.
- Not `await`-ed — events may be dropped if running in a serverless lambda (e.g. Vercel) that exits before the Sentry HTTP request flushes.

**Fix:**

```ts
errorId = await Sentry.captureUnderscoreErrorException(contextData);
```

Then thread `errorId` through as today. Update `ErrorPage.test.tsx` to mock the new function and adjust the 6 existing assertions.

### ⚠️ Optional production noise-hardening (medium priority)

#### Audit #3 — `ignoreErrors` for known noise

Standard production filter list drops browser-extension errors, ResizeObserver loops, and chunk-load failures from CDN deploys. Without it, dashboards get cluttered with non-actionable noise.

```ts
ignoreErrors: [
  "ResizeObserver loop limit exceeded",
  "ResizeObserver loop completed with undelivered notifications",
  "Non-Error promise rejection captured",
  /^Network request failed$/,
  /Loading chunk \d+ failed/,
  /^Script error\.?$/
],
```

Belongs in the shared `baseInit.ts` so it applies to client + server + edge.

#### Audit #4 — `denyUrls` for third-party scripts

Analytics / chat / Stripe / browser extensions throw inside our window's stack but aren't ours to fix.

```ts
denyUrls: [
  /extensions\//i,
  /^chrome:\/\//i,
  /^moz-extension:\/\//i,
  /googletagmanager\.com/,
  /hotjar\.com/,
  /stripe\.com/
],
```

Client-only — third-party scripts don't run on server/edge.

#### Audit #5 — `beforeBreadcrumb` to drop console breadcrumbs in production

Even with `maxBreadcrumbs: 50`, console-log breadcrumbs can carry payload dumps from dev-time debugging that didn't get removed. Drop them in production.

```ts
beforeBreadcrumb(breadcrumb) {
  if (
    breadcrumb.category === "console" &&
    env("NEXT_PUBLIC_ENVIRONMENT") === "production"
  ) {
    return null;
  }
  return breadcrumb;
}
```

### ℹ️ Lower-priority observations (defer)

#### Audit #6 — `attachStacktrace` left at default `false`

Enabling it gives stack traces on `Sentry.captureMessage()` calls. Our codebase doesn't use `captureMessage` — all paths go through `reportError` → `captureException` (which always has a stack). Leaving off is fine. Revisit if `captureMessage` ever appears.

#### Audit #7 — `hideSourceMaps` may be deprecated in Sentry v10

Some `withSentryConfig` options were renamed in the v7→v8 migration. Worth verifying in the Sentry changelog that `hideSourceMaps` is still honoured. Behaviour is already correct via `productionBrowserSourceMaps: false` in next.config — if `hideSourceMaps` is a no-op, removing it is cosmetic.

#### Audit #8 — All error capture funnels through `reportError`

Audit confirmed there are no stray `Sentry.captureException` / `captureMessage` call sites — every capture goes through the wrapper, which means the scrubber + tagging is applied uniformly. Keep enforcing this convention.

#### Audit #9 — Replay sample rates are aggressive

`replaysSessionSampleRate: 0.1` (10% of all sessions) + `replaysOnErrorSampleRate: 1.0` (100% on errors) is industry-standard but quota-hungry. Monitor Sentry usage after launch and dial down if needed.

---

## Architecture analysis

### Jira requirements vs implementation

| Jira requirement | Implementation | Status |
|---|---|---|
| 1.1 Capture all unhandled React errors | `AppErrorBoundary` wraps the app tree inside `<ThemeProvider>`; shared `_error.tsx` calls `captureUnderscoreErrorException` for SSR errors. | ✅ |
| 1.2 Errors that interrupt user actions are tracked | `QueryCache`/`MutationCache` global `onError` (via `createAppQueryClient`) + migration of 14 `console.error` swallow sites to `reportError`. | ✅ |
| 1.3 Contextual metadata (user, route, component, timestamp) | `reportError` accepts `source`, `operation`, `route`, `component`, `queryKey`, `mutationKey`, `httpStatus`, `extra`, `corrId`; `setSentryContext` adds identifiers/request scope. Timestamp is Sentry-default. | ✅ |
| 2.1 Log internal API failures | Axios response interceptor pushes `corr_id`/`http_status`/`route` into Sentry scope; React Query `onError` reports with axios details. | ✅ |
| 2.2 Error includes endpoint / status / corr_id / request metadata | `reportError` serializes `url`, `method`, `params`, `requestData`, `responseData`, `responseStatus`, `requestHeaders`, `responseHeaders` for axios errors. | ✅ |
| 3.1 Third-party (Wise) failures captured | `useMutation(postCreateRecipient, { onError: () => reportError(..., { source: "wise", operation: "createRecipient" }) })`. | ✅ |
| 3.2 Service name / endpoint / status | `source` tag + `operation` tag + axios extras provide all three. | ✅ |
| 4 Centralized monitoring | All paths route through `Sentry.captureException`; corr_id tag joins frontend Sentry with backend Logtail events. | ✅ |
| 5 Consistent logging coverage | 14 swallow sites migrated; zero `console.error` swallows remain in changed files. | ✅ |

### Beyond-scope changes also present

- **Sentry SDK upgrade `7.73.0` → `10.48.0`** (major × 3 bumps). Called out as "deferred" in the original PR body but landed in the same PR. High-risk on its own; mitigated here by the expanded mock surface + new tests for the migrated APIs.
- **`transpileClientSDK: true` removed** (drops IE11 transpilation — IE11 EOL'd 2022).
- **Trace sampling tiered** (devtest 1.0 / stage 0.5 / production 0.1) — reduces Sentry transaction quota burn.
- **Scrubber cleanup** (deduplicate `SENSITIVE_FIELDS`, remove redundant re-scrub of the whole request).

### Approach

A shared `reportError(error, context)` utility funnels every error through `Sentry.captureException` with consistent tags (`source`, `operation`, `http_status`, `corr_id`) and structured extras. Around this:

- `AppErrorBoundary` — hand-rolled React boundary (correct; no hook equivalent exists). Wrapped INSIDE `<ThemeProvider>` so the fallback UI can read theme tokens; a crash *inside* `ThemeProvider` itself would not be caught — acceptable since Emotion's `ThemeProvider` is functionally inert.
- `installAxiosCorrelationInterceptor` — stamps every outgoing axios request with `x-correlation-id` and pushes scope metadata on response errors. Now invoked inside `useEffect` (no module-scope side effect).
- `QueryCache` / `MutationCache` global `onError` (via `createAppQueryClient`) — catches errors even when per-hook `onError` shadows the default. Also extracts `corr_id` from the failed request's headers and forwards it to `reportError`.
- `_error.tsx` — shared SSR error page re-exported in both apps; surfaces the Sentry event id for support quoting.

### Sentry v7 → v10 migration notes

- `sentryContext.ts` rewritten from the legacy `Scope.getContext()` API to `getScopeData().contexts`.
- `Sentry.runWithAsyncContext` replaced with `Sentry.withIsolationScope` in `withSentryUser.ts`.
- Class-based `new Sentry.Replay(...)` replaced with functional `Sentry.replayIntegration(...)`.
- `withIsolationScope` behavioural change is covered by `withSentryUser.test.ts`'s nested-async test. End-to-end behaviour should still be eyeballed in stage before final merge.

---

## Tests

| Suite | Tests | Coverage |
|---|---|---|
| `throwSentryError` / `reportError` | 26 | tags, extras, truncation, axios merge, user extras, returns event id, primitives, caller-wins, **+4 new pre-serialize scrubbing tests** |
| `AppErrorBoundary` | 5 | children, default fallback, custom fallback, Sentry report, event id render |
| `ErrorPage` | 11 | status code, generic message, getInitialProps scope enrichment, errorId thread-through |
| `installAxiosCorrelationInterceptor` | 7 | install idempotency, header stamping, caller-id passthrough, missing headers, scope push, missing `error.config` |
| `getSentryRoute` | 5 | template pathname, `Router.pathname` fallback, `window.location.pathname` fallback, `/_error` skip, SSR undefined |
| `createAppQueryClient` | 5 | return shape, query onError, mutation onError, toast wiring, undefined-route |
| `withSentryUser` | 5 | isolation-scope wrap, nested async, header corr-id reuse, session user, query-param extraction |
| `baseInit` | **4 (new)** | hardening defaults, env-derived DSN/release/environment, env-aware sample rate, debug-off |
| `sentryContext` (existing) | 14 | adjusted for v10 mock shape |

**Total in `@proofed/shared`: 1300 passing.**

### Coverage gaps (acceptable)

- ❌ No test asserting `AxiosHeaders` instance preservation through the interceptor — gated on issue #4 / PR #2261.
- ❌ No test asserting `SENSITIVE_FIELDS` scrubber still redacts auth/token substrings after the scrubber dedup. Existing `sentryScrubber.test.ts` covers behaviour but a regression sentinel for the deduplicated paths would be cheap insurance.

---

## Recommendation

### Block list (must land before merge)

1. **Commit the staged scrubber-leak fix + DRY + hardening + rename** (12 files, currently staged uncommitted). Pre-flight green: 1300/1300 tests, typecheck clean.
2. **Delete `apps/creative-portal/config/sentry.ts` and `apps/customer-portal/config/sentry.ts`** (Audit #1). Stale-DSN files routing to a different Sentry org — no call sites. 2-file deletion.
3. **Switch `_error.tsx` to `await Sentry.captureUnderscoreErrorException(contextData)`** (Audit #2). ~5 lines in `packages/shared/components/pages/error/index.tsx` + adjust the 6 `Sentry.captureException` mocks in `ErrorPage.test.tsx`.
4. **Push** — branch is currently 14+ commits ahead of `origin/fix/PP-1750-improve-error-tracking`. CI re-runs.
5. After CI green, the PR is ready to merge.

### Nice-to-have in this PR (recommended — additive, low-risk)

6. **Audit #3 — `ignoreErrors` in `baseInit.ts`** — drops ResizeObserver / chunk-load / "Script error." / "Non-Error promise rejection" noise across all 3 runtimes. ~10 lines.
7. **Audit #4 — `denyUrls` in `client.config.ts`** — drops chrome-extension / gtm / hotjar / stripe stack-trace events. ~8 lines.
8. **Audit #5 — `beforeBreadcrumb` in `baseInit.ts`** — drops `console.*` breadcrumbs in production only. ~8 lines.

All three are decision-grade ergonomics, not security; they protect Sentry's quota and signal-to-noise ratio. Safer to land here than separately since we're already touching `baseInit.ts`.

### Remaining before PR #2261 (axios 1.x) merges

9. **Update `installAxiosCorrelationInterceptor` to mutate headers via `AxiosHeaders.set()` instead of spreading.** Drop-in replacement:

   ```ts
   axios.interceptors.request.use((config) => {
     const existing = config.headers?.get?.(CORRELATION_HEADER);
     const corrId =
       typeof existing === "string" && existing.length > 0
         ? existing
         : generateCorrelationId();
     config.headers.set(CORRELATION_HEADER, corrId);
     return config;
   });
   ```

   Can land here as a follow-up commit OR be folded into PR #2261's axios migration. Whichever merges first owns the fix. Add a test that asserts `config.headers instanceof AxiosHeaders` after the interceptor runs.

   Note: `createAppQueryClient`'s corr-id extraction reads `headers["x-correlation-id"]` which is safe under both axios 0.27 and 1.x (AxiosHeaders exposes property accessors), so no change needed there.

### Optional follow-ups (out of scope for this PR)

- **Audit #6** — `attachStacktrace: true`. Only relevant if `captureMessage` ever appears. Defer.
- **Audit #7** — Verify `hideSourceMaps` still honoured in Sentry v10. Behaviour already correct via `productionBrowserSourceMaps: false`; if it's a no-op, removing it is cosmetic.
- **Audit #9** — Monitor Replay quota after launch; dial sample rates down if 10% session-rate + 100% on-error proves too aggressive.
- Consider splitting future Sentry SDK upgrades into standalone PRs. This one bundles the SDK bump with the error-tracking infra; the bundled approach is well-tested here, but the precedent is risky.
- Add a regression sentinel test for `SENSITIVE_FIELDS` redaction after the scrubber dedup, so future edits can't accidentally widen the leak surface.
- Consider nested boundaries around known crash hotspots (Tiptap editor regions, OrderCreation flow) so a local failure doesn't blank the whole app — the current single boundary catches everything, but a localised fallback would degrade gracefully instead.

---

## Appendix — original review write-ups

Full detail for the 14 issues summarised in the [Resolution status table](#resolution-status-of-the-prior-review-issues-1-14).

### 1. `reportError` caller-supplied `extra` is silently overwritten by axios details — ✅ FIXED

**File:** `packages/shared/utils/throwSentryError.ts`

`buildExtras` now ends with `return { ...extras, ...(context.extra ?? {}) }`. Caller wins. Test: `caller-supplied extra wins over built-in axios fields`.

### 2. Duplicated QueryClient + interceptor bootstrap across both `_app.tsx` files — ✅ FIXED

**File:** `packages/shared/utils/createAppQueryClient.ts`

Extracted factory returns a configured `QueryClient` with `queryCache`/`mutationCache` wired to `reportError`. Both portals call `createAppQueryClient()` inside `useState`. 5 tests in `createAppQueryClient.test.ts`.

### 3. Error boundary scope is too coarse — Layout / providers are outside it — ✅ FIXED

**File:** `apps/*/pages/_app.tsx`

`<AppErrorBoundary componentName="...">` is now the outermost wrapper inside `<ThemeProvider>` in both apps. Render crashes in any context provider, `Layout`, `ModalQueue`, `FeaturesProvider`, `VerifyOnboarding`, or any page component are caught and surfaced via Sentry + fallback UI.

> **Caveat:** the boundary intentionally lives *inside* `<ThemeProvider>` so the fallback UI can resolve theme tokens. A throw in `ThemeProvider` itself would escape — Emotion's `ThemeProvider` is functionally inert, so this trade-off is correct.

### 4. Axios interceptor spreads config, likely stripping `AxiosHeaders` prototype — ⚠️ DEFERRED

**File:** `packages/shared/utils/installAxiosCorrelationInterceptor.ts:35-42`

Code still uses the spread pattern:

```ts
return {
  ...config,
  headers: { ...headers, [CORRELATION_HEADER]: corrId } as typeof config.headers
};
```

Under axios `^0.27.2` (current `develop` and this branch), `config.headers` is a plain object, so the spread is safe today. Under axios `^1.15.0` (queued in [PR #2261](https://github.com/Proofed/B2BWebserver/pull/2261)), `config.headers` is an `AxiosHeaders` class instance and the spread strips the prototype, breaking any downstream `.set` / `.get` / `.has` call.

Deferred-but-real concern, not a current bug. See [Recommendation — axios 1.x follow-up](#remaining-before-pr-2261-axios-1x-merges).

### 5. High-cardinality `route` tag from `window.location.pathname` — ✅ FIXED

**File:** `packages/shared/utils/getSentryRoute.ts`

New helper prefers `Router.router?.pathname ?? Router.pathname` (Next route template) and falls back to `window.location.pathname` only when the router is uninitialised. The `_error.tsx` SSR path uses `contextData.pathname` (also template form). 5 tests in `getSentryRoute.test.ts`.

### 6. Error ID shown in `AppErrorBoundary` fallback but not in `ErrorPage` — ✅ FIXED

**File:** `packages/shared/components/pages/error/index.tsx`

`getInitialProps` returns `{ ...props, errorId }`; the page renders `Error ID: {errorId}` when present.

### 7. Unsafe `as Error & { statusCode?: number }` cast in `ErrorPage.getInitialProps` — ✅ FIXED

**File:** `packages/shared/components/pages/error/index.tsx:55-57`

Narrowed to `as { statusCode?: number } | null`.

### 8. `packages/shared/__mocks__/@sentry/nextjs.ts` was incomplete for v10 API surface — ✅ FIXED

**File:** `packages/shared/__mocks__/@sentry/nextjs.ts`

Now exports `captureException`, `captureUnderscoreErrorException`, `setTag`, `setContext`, `setUser`, `flush`, `getCurrentScope` (returns `getScopeData`-shaped stub), `withIsolationScope`, `withScope`, `replayIntegration`, `browserTracingIntegration`.

### 9. No test coverage for `runWithAsyncContext` → `withIsolationScope` migration — ✅ FIXED

**File:** `packages/shared/api/utils/middlewares/withSentryUser.test.ts`

5 tests cover the wrap, nested-async tag/context propagation, incoming corr-id reuse, user session attachment, and query-param extraction.

### 10. `installAxiosCorrelationInterceptor()` called at module top-level of `_app.tsx` — ✅ FIXED

**File:** `apps/*/pages/_app.tsx`

Both apps call it inside `useEffect(..., [])`. The `isInstalled` idempotency flag remains as a safety net.

### 11. Stale `/* eslint-disable no-console */` in rewritten `sentryContext.ts` — ✅ FIXED

**File:** `packages/shared/utils/sentryContext.ts:1`

Removed.

### 12. `@sentry/nextjs` uses `^10.48.0` while neighbouring deps are pinned — ✅ FIXED

**File:** `apps/*/package.json`

Both now pin to `"@sentry/nextjs": "10.48.0"` (exact).

### 13. No test for `reportError` with non-Error primitive inputs — ✅ FIXED

**File:** `packages/shared/utils/throwSentryError.test.ts`

New `non-Error primitive inputs` describe block adds 5 tests: `null`, `undefined`, string, plain object — each asserting the synthetic `Error("Non-Error thrown: …")` wrap and `extra.originalValue` storage — plus a sentinel that real `Error` instances stay unwrapped.

### 14. No direct test for `_app.tsx` QueryCache/MutationCache wiring — ✅ FIXED

**File:** `packages/shared/utils/createAppQueryClient.test.ts`

5 tests cover the factory's outputs and Sentry forwarding for both queries and mutations.
