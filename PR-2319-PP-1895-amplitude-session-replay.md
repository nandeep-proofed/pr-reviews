# PR Review: PP-1895: connect Amplitude and enable Session Replay

**PR:** https://github.com/Proofed/B2BWebserver/pull/2319
**Jira:** https://proofed.atlassian.net/browse/PP-1895
**Status:** In Progress

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Install + initialize Amplitude Browser SDK on app load | `@amplitude/analytics-browser` added; `initAmplitude` called once via `useAmplitude` in `_app.tsx` | ✅ |
| Set `environment` property so data can be filtered per environment | `Identify().set("environment", environment)` from `NEXT_PUBLIC_ENVIRONMENT` | ✅ |
| Install + register Session Replay plugin | `@amplitude/plugin-session-replay-browser` added; `enableSessionReplay` registers it | ✅ |
| Session Replay at 40% sample rate (≤1k/mo) | Rate from `NEXT_PUBLIC_AMPLITUDE_REPLAY_SAMPLE_RATE` (40% prod / 5% dev per Nicola) | ✅ |
| Record only on `/orders` + `/partners` incl. all sub-pages; exclude all else via allowlist | `isReplayAllowedPath` (exact + `prefix/`); replay toggled on route change | ✅ |
| `Application Loaded` verification event on initial load | `amplitude.track("Application Loaded", { environment })` | ✅ |
| Event includes `environment` property | Passed as event property | ✅ |
| Out of scope: no business events, no extra user props, no backend | Only `Application Loaded` + default autocapture; no backend changes | ✅ Respected |

Confirmed in Jira comments (Nicola #58406): dev = "default" project / prod = separate "Prod" project (per-env API key); dev sample rate ~5%; sub-pages of both sections included; autocapture limited to Sessions + Page Views. All four reflected in the implementation.

Beyond ticket scope (reasonable hardening): server zone hardcoded to `US` (client-confirmed), `env.js` registration of the new vars, and removal of `TASKS_COMPLETED.md`.

---

## Architecture Analysis

Clean separation between an SDK-facing pure-function util (`amplitude.ts`) and a thin React lifecycle hook (`useAmplitude.ts`), wired into `_app.tsx` alongside the existing `useHotjarWithIdentity` / `useSentryIdentity` hooks — consistent with the repo's analytics-integration pattern.

Init runs once, guarded by `isInitialized` ref. Session Replay is toggled per-route via an `isReplayActive` ref, so the plugin is added/removed only on allowlist transitions. The integration hard no-ops when `NEXT_PUBLIC_AMPLITUDE_API_KEY` is unset (mirrors Sentry's `enabled: !!env(...)` guard), and all config flows through `next-runtime-env` (one build, per-environment values).

Regression surface is small: two new isolated files in `@proofed/shared`, one additive hook call + import in `_app.tsx`, two new deps in `packages/shared/package.json`. The only signature change (`initAmplitude` dropping `serverZone`) is internal — sole consumers are the hook and its tests.

---

## Issues Found

### 1. Amplitude env vars registered as required in `env.js` — RESOLVED

**[File: apps/creative-portal/env.js | Function: creativePortalEnv.runtime | Severity: low]**

**Problem:** The two Amplitude vars used `envValidator(runtimeParams)` (`optional: false`), making them required.

**Impact:** With `LOCAL_ENVIRONMENT` set, validation would throw if either value was empty — contradicting the code's "no-op when API key unset" design.

**Fix:** Changed both to `optional: true` (commit `04d314b1d`). Resolved.

### 2. `enableSessionReplay(0)` registers a no-op plugin on allowed routes

**[File: packages/shared/hooks/useAmplitude.ts | Function: useAmplitude (pathname effect) | Severity: low]**

**Problem:** `replaySampleRate` defaults to `0`. On an allowlisted route with rate `0`, `enableSessionReplay(0)` still adds the replay plugin (which records nothing) and flips `isReplayActive` to true.

**Impact:** Harmless — no recordings produced, no quota consumed. Minor wasted work.

**Fix (deferred):** Optionally gate on a positive rate, e.g. `if (shouldRecord && replaySampleRate > 0 && !isReplayActive.current)`.

### 3. Malformed sample-rate env value yields `NaN`

**[File: packages/shared/hooks/useAmplitude.ts | Function: module scope | Severity: low]**

**Problem:** `Number(env(...) ?? "0")` returns `NaN` for a non-numeric value (e.g. `"40%"`).

**Impact:** `NaN` would be passed as `sampleRate`. Low likelihood (deploy-controlled value).

**Fix (deferred):** `const rate = Number.isFinite(parsed) ? parsed : 0;`

---

## Tests

- ✅ New code is tested: `utils/amplitude.test.ts` (9) — allowlist edge cases (roots, nested, prefix-collisions like `/orders-archive`), autocapture config, environment tagging, verification event, replay enable/disable.
- ✅ `hooks/useAmplitude.test.ts` (5) — init-once, enable-on-allowed-route with sample rate, no-record on excluded routes, stop-on-navigation-away, no-op when key unset.
- ✅ `npx turbo run test`: 1242/1243 pass. The one failure (`formatWordQuantity`) is a pre-existing `en-IN` locale artifact — passes under CI's `en-US`; unrelated to this PR.
- ✅ `npx turbo run typecheck`: 0 errors across all workspaces.
- ✅ `npx turbo run build`: clean (creative-portal build completed).
- ⚠️ `npx turbo run lint`: fails in 4 workspaces — all pre-existing `prettier/prettier` debt in untouched files. customer-portal & wysiwyg-editor fail too, though this PR touches neither; no Amplitude file appears in the errors. Pre-existing `develop` debt, not a regression introduced here.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ✅ |
| Code quality | ✅ |
| Mergeable state | ✅ Clean — no validation failure is attributable to this PR |

---

## Recommendation

**Approve with suggestions**

1. Implementation faithfully meets all PP-1895 requirements and the clarifications confirmed with Nicola (#58406), with good test coverage and low regression risk.
2. Validation is clean for this PR — typecheck and build pass, tests pass bar one pre-existing locale artifact, and lint failures are pre-existing `develop` debt in untouched files (track separately; do not fix inside this feature PR).
3. Issue #1 resolved (`env.js` vars now optional). Issues #2 and #3 are harmless low-priority polish — safe to defer.
4. Non-code: confirm `NEXT_PUBLIC_ENVIRONMENT` is set on creative-portal devtest/stage/production with canonical values (drives Amplitude's environment tag; already required by Sentry).
