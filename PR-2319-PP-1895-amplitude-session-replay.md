# PR Review: PP-1895 — connect Amplitude and enable Session Replay

**PR:** https://github.com/Proofed/B2BWebserver/pull/2319

**Jira:** https://proofed.atlassian.net/browse/PP-1895

**Status:** Code Review (Jira) · open · `mergeable_state: dirty` (GitHub) · base `develop` (`b1932a37`) · head `4cfef246` · +709/−682 · 10 files

_Consolidated review — supersedes the two earlier drafts (author self-review + independent re-review) for this PR._

---

## Scope

Reviewable source (excluded `yarn.lock` — checked via the new-dependency pass below — and the deleted `TASKS_COMPLETED.md` scratch doc):

- `packages/shared/utils/amplitude.ts` (new) — SDK init + replay enable/disable + `isReplayAllowedPath`
- `packages/shared/hooks/useAmplitude.ts` (new) — lifecycle hook
- `packages/shared/utils/amplitude.test.ts` (new, 9 tests) · `packages/shared/hooks/useAmplitude.test.ts` (new, 5 tests)
- `apps/creative-portal/pages/_app.tsx` — wires `useAmplitude()`
- `apps/creative-portal/env.js` · `.env.example` — registers two optional env vars
- `packages/shared/package.json` — adds `@amplitude/*` deps

Lenses run: correctness, regressions/shared-API, type-safety, React/perf, reuse/conventions, security/PII, error-handling/observability, tests, new-dependency. Accessibility: **n/a** (no UI/markup changed — only a hook call added).

---

## Jira Requirements vs Implementation

Mapped against the PP-1895 acceptance criteria **and** the clarification thread (Nicola, comment #58406: default project for dev @ 5%, dedicated Prod project @ 40%, subpages included, Sessions + Page Views auto-capture only).

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| Install `@amplitude/analytics-browser` | Added to `packages/shared/package.json` (`2.42.5`) + `yarn.lock` | ✅ Addressed |
| Initialise SDK on app load | `initAmplitude` called from `useAmplitude` init effect, wired into `_app.tsx` | ✅ Addressed |
| Set `environment` property so data is filterable per env | `amplitude.identify` sets `environment`; also passed on the load event | ✅ Addressed |
| Initialise in production (key via config, not hardcoded) | Reads `NEXT_PUBLIC_AMPLITUDE_API_KEY` via `next-runtime-env` (better than the hardcoded key in the ticket) | ✅ Addressed |
| Install + register Session Replay plugin | `@amplitude/plugin-session-replay-browser` (`1.31.0`) added; registered in `enableSessionReplay` | ✅ Addressed |
| 40% sample rate (≤1k sessions/mo); 5% dev (#58406) | `NEXT_PUBLIC_AMPLITUDE_REPLAY_SAMPLE_RATE`, set per environment | ✅ Addressed (config-driven) |
| Record only on `/orders` + `/partners` incl. all subpages; allowlist all else | `isReplayAllowedPath` prefix match (`=== prefix \|\| startsWith(prefix + "/")`); toggled on route change | ✅ Addressed |
| `Application Loaded` event on load with `environment` | `amplitude.track("Application Loaded", { environment })` | ✅ Addressed |
| Auto-capture: Sessions + Page Views only (#58406, PII-minimal) | `autocapture` enables `sessions`/`pageViews`, disables form/element/network/attribution/fileDownloads | ✅ Addressed |
| Conservative masking (no PII) | `privacyConfig.defaultMaskLevel: "conservative"` | ✅ Addressed |
| No-op when API key unset | Both effects early-return on empty `apiKey` | ✅ Addressed |
| Out of scope: no business events, no extra user props, no backend | None added | ✅ Respected |
| SDK must not add >50 ms to page load | Not measured in this PR (ticket checklist marks this ❌) | ⚠️ Unverified — see Open Questions |

No code scope creep. The only out-of-band change is the deletion of `TASKS_COMPLETED.md` (a stray tracking doc) — harmless.

---

## Architecture Analysis

Clean separation: a pure, SDK-facing utility (`amplitude.ts`) exposing `initAmplitude` / `enableSessionReplay` / `disableSessionReplay` / `isReplayAllowedPath`, plus a lifecycle hook (`useAmplitude.ts`) that owns init-once + route-based replay toggling. Refs guard against re-init and redundant replay toggling. This mirrors the existing `useHotjarWithIdentity` pattern (module-level env constants + `useEffect`), so it fits repo convention.

Purely additive — new shared exports, **no existing shared export signature changed**, so no cross-app breakage. The hook lives in `packages/shared` but its only importer is creative-portal `_app.tsx` (confirmed: it's the sole file in the PR's changed-file list that imports `useAmplitude`, and no other file in the repo references it); customer-portal does not import it, so tree-shaking keeps the Amplitude deps out of that bundle. Regression risk is **low**.

**New dependencies:** `@amplitude/analytics-browser@2.42.5` and `@amplitude/plugin-session-replay-browser@1.31.0` — both required by the ticket, first-party Amplitude packages, exact-pinned. No conflict with the React 18.2.0 / Tiptap 3.10.7 resolutions.

**Server zone hardcoded to `US`** — documented in the PR as confirmed with the client; acceptable.

---

## Issues Found

### 1. Merge conflicts with `develop` — cannot merge

**[File: (PR-level)]**

**Function/Class:** —

**Severity:** blocker

**Confidence:** high

**Problem:** GitHub reports the PR as conflicting with the base branch.

**Evidence:** PR payload `mergeable_state: "dirty"` (equivalently `mergeable: CONFLICTING` / `mergeStateStatus: DIRTY`).

**Impact:** The PR cannot be merged as-is; conflicts must be resolved and CI re-run on the new tip. Mechanical, but a hard merge gate.

**Fix:** Rebase/merge latest `develop` into the branch, resolve conflicts, push, and re-validate.

### 2. Replay sample rate is coerced with `Number()` and never numerically validated

**[File: packages/shared/hooks/useAmplitude.ts]**

**Function/Class:** module-scope `replaySampleRate` → `enableSessionReplay`

**Severity:** medium

**Confidence:** high

**Problem:** `const replaySampleRate = Number(env("NEXT_PUBLIC_AMPLITUDE_REPLAY_SAMPLE_RATE") ?? "0")` accepts any string and passes it straight into `enableSessionReplay(replaySampleRate)`. The `env.js` validator for this key is `envValidator({ min: 1, max: 200, optional: true })`, and `envValidator` (`packages/shared/utils/env-utils.js`) only checks **string length** (1–200 chars) — not numeric format or range. So: a malformed value (`"abc"`, `"40%"`) deploys cleanly and yields `sampleRate: NaN`; a percentage-style value (`"40"` intending 40%) yields `40`, far outside Amplitude's expected `0..1` fraction range.

**Evidence:** `useAmplitude.ts` coercion `const replaySampleRate = Number(env(...) ?? "0");` and the replay effect `enableSessionReplay(replaySampleRate)`. `env-utils.js` `envValidator` optional branch: `if (min && stringValue.length < min)` / `if (max && stringValue.length > max)` — length only, no numeric parse. (Note: the earlier author self-review's worry that this would *fail env validation* is unfounded — it's length-only, so bad values pass through silently.)

**Impact:** A single-character env typo silently breaks the observability feature with no error surfaced: `NaN` → undefined sampling; `"40"` → effectively always-record, which can exhaust the shared **1k-sessions/month** Session Replay quota the ticket is explicitly trying to protect. Correct fraction values behave correctly, so the happy path is fine.

**Fix:** Clamp/validate the parsed rate to `0..1`, degrading a bad value to "no recording", and add a test:

```typescript
const parsedRate = Number(
  env("NEXT_PUBLIC_AMPLITUDE_REPLAY_SAMPLE_RATE") ?? "0"
);
const replaySampleRate =
  Number.isFinite(parsedRate) && parsedRate >= 0 && parsedRate <= 1
    ? parsedRate
    : 0;
```

Add a `.env.example` comment noting the value is a `0..1` fraction.

### 3. `enableSessionReplay(0)` registers a no-op plugin on allowlisted routes

**[File: packages/shared/hooks/useAmplitude.ts]**

**Function/Class:** `useAmplitude` replay-toggle effect / `enableSessionReplay`

**Severity:** low

**Confidence:** high

**Problem:** When the sample rate is `0` (the documented "0 = no recording, frees the shared quota" case), navigating onto `/orders`/`/partners` still calls `enableSessionReplay(0)`, running `amplitude.add(sessionReplayPlugin({ sampleRate: 0, ... }))` — registering an inert plugin that is then removed again on the next excluded-route navigation.

**Evidence:** replay effect `if (shouldRecord && !isReplayActive.current) { enableSessionReplay(replaySampleRate); ... }` — no `replaySampleRate > 0` guard; `enableSessionReplay` always calls `amplitude.add(...)`.

**Impact:** Cosmetic/negligible — redundant add/remove of an inert plugin; no recording, no correctness impact. Called out in the PR body as a known deferral.

**Fix (optional):** Gate registration on a positive rate — `if (shouldRecord && replaySampleRate > 0 && !isReplayActive.current)` (mirror in the disable branch). Folds naturally into the fix for #2.

### 4. Session-replay plugin is statically imported, so it ships on every route

**[File: packages/shared/utils/amplitude.ts]**

**Function/Class:** module import / `enableSessionReplay`

**Severity:** low

**Confidence:** high

**Problem:** `sessionReplayPlugin` is a top-level static import (`import { sessionReplayPlugin } from "@amplitude/plugin-session-replay-browser"`). Because `amplitude.ts` is pulled in by `useAmplitude`, which is mounted in `_app.tsx`, the session-replay bundle is included in the main creative-portal chunk and loaded on **every** route — even the majority (`/`, `/dashboard`, `/settings`, …) where replay never activates.

**Evidence:** `amplitude.ts:2` static import; used only inside `enableSessionReplay` (`amplitude.ts:65`), which itself only runs on `/orders` and `/partners`.

**Impact:** The session-replay package (non-trivial in size) is parsed/downloaded on all pages, which works against the ticket's explicit **≤ 50 ms page-load budget** (Issue in Open Questions). No functional impact.

**Fix (optional):** Lazy-load the plugin only when replay is actually enabled, e.g.:

```typescript
export const enableSessionReplay = async (sampleRate: number) => {
  const { sessionReplayPlugin } = await import(
    "@amplitude/plugin-session-replay-browser"
  );
  amplitude.add(
    sessionReplayPlugin({
      sampleRate,
      privacyConfig: { defaultMaskLevel: "conservative" }
    })
  );
};
```

Keeps the replay bundle off routes that never record and tightens the page-load budget.

### 5. Unset `NEXT_PUBLIC_ENVIRONMENT` silently tags events as `""`

**[File: packages/shared/hooks/useAmplitude.ts]**

**Function/Class:** module-scope `environment`

**Severity:** low (marginal — largely mitigated by existing env validation)

**Confidence:** high

**Problem:** `const environment = env("NEXT_PUBLIC_ENVIRONMENT") ?? ""` (`useAmplitude.ts:14`) — if the var is unset, every event and the `environment` user property are tagged with an empty string rather than a sensible fallback, undermining the ticket's per-environment filtering.

**Mitigating fact (verified):** `env.js:33` already declares `NEXT_PUBLIC_ENVIRONMENT: envValidator(runtimeParams)` with `runtimeParams.optional = false` — i.e. **required**. So in any deploy where runtime env validation runs, an unset value fails fast at startup and never reaches this fallback. The empty-string path only materialises if that validation is bypassed (it's gated on `process.env.LOCAL_ENVIRONMENT`). This makes the finding largely theoretical.

**Evidence:** `useAmplitude.ts:14` `environment` constant; passed to `initAmplitude` → `identify.set("environment", "")` and `track("Application Loaded", { environment: "" })`. `env.js:33` marks the var required (`optional: false`). The PR's own deploy note also asks to confirm `NEXT_PUBLIC_ENVIRONMENT` is set.

**Impact:** Data-quality only, and only in the unlikely event validation is skipped — events would land under a blank environment in Amplitude. No user impact.

**Fix (optional):** A labelled fallback (`?? "unknown"`) would make a bypassed-validation misconfiguration visible in the data rather than silent — nice-to-have, not necessary given the required-env declaration.

---

## Open Questions

- **Page-load performance budget:** the ticket requires the SDK add **≤ 50 ms** to page load, and the ticket's own checklist leaves that item unchecked (❌). This PR includes no before/after measurement. Was DevTools Performance captured out-of-band, or should it be done before merge? — `apps/creative-portal/pages/_app.tsx` (`useAmplitude()` call site).
- **Module-level `env()` timing:** `apiKey` / `environment` / `replaySampleRate` are read at module import, not inside the effect. `next-runtime-env` reads at runtime and this mirrors `useHotjarWithIdentity`, so it's very likely fine on the client — but confirm the public env is injected before this module is first imported during hydration (couldn't verify without running). — `packages/shared/hooks/useAmplitude.ts`.
- **Cookie consent / GDPR:** there is no in-app consent gate (no OneTrust/Cookiebot/`cookieconsent` anywhere in the repo), so Amplitude + Session Replay start on load for all users. This is **consistent with the existing `useHotjarWithIdentity` hook**, which already runs session-recording analytics with the same env-only gating — so it's not a regression this PR introduces. Confirm consent for analytics/session-recording is handled outside the app (infra/legal/legitimate-interest) rather than needing an in-app gate. — app-wide, not specific to this PR.

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ Not run | User declined the validation run this session — static-only review |
| `npx turbo run typecheck` | ⏭️ Not run | " |
| `npx turbo run lint` | ⏭️ Not run | " |
| `npx turbo run build` | ⏭️ Not run | " |

The validation suite was **not run** in this review. The PR description records a prior run against an earlier commit (`04d314b1d`): typecheck ✅ / build ✅ / test ✅ (1242/1243 — the single failure a pre-existing `en-IN` locale artifact in `formatWordQuantity`, unrelated) / lint ⚠️ pre-existing `develop` prettier debt in untouched files, no Amplitude file involved. That run predates head `4cfef246` — re-run `test` / `typecheck` / `lint` / `build` (**full monorepo**, since `packages/shared` is touched) on the current tip before merge (CLAUDE.md hard gate).

---

## Tests

- ✅ Strong coverage: 9 util tests + 5 hook tests, asserting real behaviour (exact `init` config, `environment` identify, `Application Loaded` payload, conservative masking + sample rate, plugin removal by name, init-once, enable/disable on navigation, no-op when key unset).
- ✅ Genuinely adversarial allowlist cases: roots, nested subpages, excluded routes, and prefix-collision traps (`/orders-archive`, `/partnerships`) that a naive `startsWith` would wrongly match.
- ⚠️ **Gap:** no test for a malformed / `NaN` / out-of-range sample rate (ties to Issue #2) or the `sampleRate === 0` no-op path (Issue #3). Add with the fix.
- ⚠️ The hook test re-implements `isReplayAllowedPath` inline rather than importing the real util (documented as intentional; the util is separately, thoroughly tested). Acceptable — minor drift risk if the real allowlist changes.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ Meets all confirmed requirements; Issue #2 (medium) unvalidated env coercion |
| Regression risk | ✅ Low — additive; single consumer; no shared export signature changed |
| Tests | ✅ Good coverage & quality (one edge-case gap) |
| Accessibility | n/a — no UI/markup changed |
| Error handling | ⚠️ Unvalidated sample-rate coercion (Issue #2) |
| Security / PII | ✅ Public client key via env (not committed); conservative masking; PII-minimal autocapture. Low-risk telemetry change — a full `/security` pass is optional. |
| Code quality | ✅ Clean util + hook split; mirrors `useHotjarWithIdentity` |
| New dependencies | ✅ `@amplitude/*` exact-pinned; required for the feature |
| Validation suite | ⏭️ Not run this session (user opted out) — re-run before merge |
| Mergeable state | ❌ Dirty — conflicts with `develop` (Blocker) |

---

## Recommendation

**Request changes** — the code is close to mergeable and low-risk, but an open Blocker (merge conflict) forces this per the severity rubric.

1. **Blocker —** resolve the `develop` merge conflict and re-run CI on the new tip (Issue #1).
2. **Should-fix (medium) —** clamp/validate the replay sample rate to `0..1` so a malformed or percentage-style env value can't produce `NaN` / over-record and blow the 1k/month quota; add a test and a `.env.example` note (Issue #2).
3. **Optional (low) —** skip plugin registration when the rate is `0` (Issue #3); lazy-`import` the session-replay plugin so it doesn't ship on every route (Issue #4); give `environment` a labelled fallback (Issue #5).
4. **Re-run the validation suite** (full monorepo) on head `4cfef246` before merge — the recorded run predates the current tip.
5. **(Low) Confirm the ≤50 ms page-load budget** was measured (Issue #4 works against it), or capture it (Open Questions).
6. **(Info) Confirm analytics/session-recording consent** is covered outside the app (consistent with existing Hotjar; Open Questions).

Once the conflict is resolved and Issue #2 is addressed, this is a clean, well-tested, idiomatic, low-risk addition.
