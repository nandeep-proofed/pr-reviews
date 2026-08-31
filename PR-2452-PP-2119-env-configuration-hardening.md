# PR Review: chore/PP-2119: Environment configuration hardening

**PR:** https://github.com/Proofed/B2BWebserver/pull/2452
**Jira:** https://proofed.atlassian.net/browse/PP-2119
**Status:** In Progress
**Branch:** `chore/PP-2119-env-configuration-hardening` → `develop`
**Head reviewed:** `5125fe9ff6f7e14486db243a0bae3b5ea8b76104` (base `e5e016eec`)
**Size:** 23 files, +1467 / −191

---

## What this means for users (non-technical summary)

There is no UI in this change. The people affected are developers, and — if the
open questions below go the wrong way — everyone using the live site.

1. **Anyone who pulls this branch and follows the setup instructions cannot start
   the app.** The setup template ships one required field blank and gives no hint
   what to put in it. The instructions in the PR tell you to set one *other*
   field, so following them exactly still leaves you with a server that refuses
   to start. This hits every reviewer of this PR and every new joiner.
2. **The safety net has an off switch that works by accident.** The change adds
   guards meant to stop a developer-only setting from reaching a live server. If
   the environment name on that server is misspelled — `Production` instead of
   `production` — and a troubleshooting flag is left on, every guard silently
   switches off, including the one that keeps two-factor login turned on.
3. **The headline safety property is never tested.** The change claims a
   look-alike web address cannot impersonate a real backend. We broke that
   protection deliberately and the whole test suite still passed, so if someone
   weakens it later nothing will notice.
4. **Most of the new tests for one portal don't actually check anything.** We
   switched the entire validation feature off and 18 of that portal's 19 tests
   still reported success.
5. **A test server that has two-factor login turned off will now refuse to
   start**, and the documented way to force it past that is refused too. Whether
   this bites depends on how the test servers are currently configured — see
   Open Questions.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Make the Test backend reachable from a local dev server | `SERVICES_URI_REWRITE` + new `rewriteServiceUri.ts`; applied in `getAll.ts` to the directory URL and each `service.uri` | ✅ Addressed |
| Rewrite must be identity when unset (deployed behaviour byte-identical) | `parseServiceUriRewrites("")` → `[]`; `rewriteServiceUri` short-circuits at `rewriteServiceUri.ts:117-119` | ✅ Verified |
| Match on the parsed host so a look-alike suffix cannot spoof | Exact `===` on lowercased `URL.host` (`rewriteServiceUri.ts:129-131`) | ⚠️ Correct in code, **untested** — see Issue 3 |
| Audit all 30 env keys against real usage; remove dead keys | 4 of the 5 named keys genuinely removed and verified unreferenced | ⚠️ Partial — see Issue 20 |
| Validation must actually run (was decorative) | `next.config.js` (build group) + `instrumentation.ts` (all groups); `instrumentationHook: true` confirmed at `packages/shared/scripts/nextConfig.js:55-57` | ✅ Addressed |
| Validate shape, not string length | yup schemas with `oneOf` / `httpUrl()` / `number()` per field | ✅ Addressed |
| Report every problem at once | `abortEarly: false` + aggregation (`env-utils.js:161-177`) | ✅ Addressed |
| Never echo a value in an error message | Every field overrides its message; confirmed across both schemas × 19 canaries, 0 echoes | ✅ Verified |
| Fail closed rather than serving 500s | `process.exit(1)` in `instrumentation.ts:31` | ⚠️ Works, but see Issues 2, 5, 10 |
| Guard **every** deployed environment against a stray rewrite / `MFA_DISABLED` | `DEPLOYED_ENVIRONMENTS` includes `devtest` | ⚠️ Bypassable — see Issues 2, 9 |
| `.env.example` for every `.env` | Root, creative, customer all present, all values empty | ✅ Verified — no secrets committed, branch history checked |
| yup declared where imported | `packages/shared/package.json:70`, `apps/customer-portal/package.json:34`; `yarn.lock` genuinely unchanged | ✅ Verified |

**Scope creep:** none. Every changed file serves the ticket. The PR is large but
single-purpose.

---

## Architecture Analysis

The approach is sound and the code is unusually well-commented — most comments
explain *why*, which is rare and genuinely valuable here. Three things stand out
as good judgement calls:

- Building on **yup** rather than a bespoke validator matches the repo's existing
  convention (139 files) and avoids inventing a second validation vocabulary.
- Keeping schemas in each app's `env.js` and only the shared machinery in
  `env-utils.js` is the right split — it stops the two portals drifting without
  forcing a shared schema neither app fully wants.
- Deliberately **not** using yup's `.url()` (`env-utils.js:52-56`) is correct;
  that matcher rejects `https://localhost:3000` and accepts `javascript:`.

The design weakness is structural rather than local: **every guard in this system
keys off `NEXT_PUBLIC_ENVIRONMENT`, which is itself just another line in the same
`.env` file the guards are protecting.** The realistic accident this PR exists to
prevent is "someone copied a working laptop `.env` onto a server" — and in
exactly that accident, `NEXT_PUBLIC_ENVIRONMENT=local` travels along with
`SERVICES_URI_REWRITE`, so nothing fires. That does not make the change worse
than what is on `develop` today (which validates nothing at all), but the
comments claim a stronger property than the code delivers.

The second structural point: this converts a large set of previously-soft
failures into **process exits**. That is the right direction, but it means the
first deploy after merge is the moment every latent env mismatch across the fleet
surfaces simultaneously, and the repo contains no evidence of what those VMs
actually hold.

---

## Issues Found

### 1. A fresh `.env.example` copy cannot start either portal

**[File: apps/creative-portal/.env.example:42, apps/customer-portal/.env.example:41]**

> **In plain terms:** Following the project's own setup instructions leaves you
> with an app that refuses to start. The setup template has one required field
> left blank with no note saying what to put there, and the app now treats a
> blank there as a fatal error. Every reviewer of this change and every new
> joiner hits this on their first run.

**Function/Class:** `validateCreativePortalEnv` / `validateCustomerPortalEnv` (build group)

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. Check out the branch on a machine with no existing `.env`.
2. `cp apps/creative-portal/.env.example apps/creative-portal/.env`
3. Fill in the values, including `NEXT_PUBLIC_ENVIRONMENT=local` exactly as the
   PR's "Action required for reviewers" section instructs.
4. `yarn dev:creative-portal`
5. **Expected:** the dev server starts.
6. **Actual:** it exits before Next boots with
   `[creative-portal] environment validation failed with 1 problem(s): - NEXT_PUBLIC_APP_VERSION is required [build]`

**Problem:** `.env.example` ships `NEXT_PUBLIC_APP_VERSION=` with no value and no
guidance, while `next.config.js:11` now runs `validate*PortalEnv({ groups: ["build"] })`
and the build group declares it `.required()`. `blankAsMissing`
(`env-utils.js:149-159`) converts `""` to `undefined`, so the blank template value
trips `.required()`.

**Evidence:** `apps/creative-portal/env.js:80-82`:

```js
  build: {
    NEXT_PUBLIC_APP_VERSION: Yup.string().required("is required")
  },
```

Run against the real schema with a fresh-template environment (only
`NEXT_PUBLIC_ENVIRONMENT=local` filled in, as the PR instructs):

```
--- next.config.js path: groups=[build] ---
THREW:
[creative-portal] environment validation failed with 1 problem(s):
  - NEXT_PUBLIC_APP_VERSION is required [build]

--- instrumentation.ts path: all groups ---
THREW:
[creative-portal] environment validation failed with 2 problem(s):
  - NEXT_PUBLIC_APP_VERSION is required [runtime]
  - NEXT_PUBLIC_APP_VERSION is required [build]
```

Confirmed that `.env` is loaded before `next.config.js` is required
(`next/dist/server/config.js:675` calls `loadEnvConfig` before `require(path)` at
`:701`), so the blank template value is what reaches the validator.

**Impact:** The documented onboarding path is broken for both portals. It also
breaks `yarn test:e2e`, because `apps/*/playwright.config.ts:14-18` starts the
suite with `command: yarn run dev --port 3000`, which loads `next.config.js`.
Contrast `NEXT_PUBLIC_ENVIRONMENT`, which the same template documents carefully
at lines 37-40 — this key got none of that treatment.

**Fix:** Give the template a working default and a comment, and consider whether
the build group should require it at all given `nextConfig.js:88-90` already
treats it as optional:

```bash
# App version, surfaced in Sentry releases. Any non-empty string works
# locally; CI supplies the real value.
NEXT_PUBLIC_APP_VERSION=0.0.0-local
```

---

### 2. `SKIP_ENV_VALIDATION` plus a misspelled environment name disables every guard on a live server

**[File: packages/shared/utils/env-utils.js:196-219]**

> **In plain terms:** The change adds safety checks that stop developer-only
> settings reaching a live server — including one that keeps two-factor login
> switched on. Those checks can be turned off entirely by a combination that is
> easy to reach by accident: a troubleshooting flag left switched on, plus the
> server's environment name written with the wrong capitalisation. The check that
> would have caught the misspelling is itself one of the checks being skipped.

**Function/Class:** `validateEnvConfig`

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. On a deployed server, set `SKIP_ENV_VALIDATION=1` (the documented
   troubleshooting flag) and `NEXT_PUBLIC_ENVIRONMENT=Production` — capital P, or
   `prod`, or leave it unset entirely.
2. Also have `MFA_DISABLED=true` and a `SERVICES_URI_REWRITE` value present, as a
   copied laptop `.env` would.
3. Boot the server.
4. **Expected:** the deployed guards reject both values, or at minimum the
   unrecognised environment name is rejected.
5. **Actual:** the server boots. Nothing is validated. Two-factor login is off
   and backend traffic carrying the `api-key` header is redirected.

**Problem:** The skip decision is taken from `NEXT_PUBLIC_ENVIRONMENT` *before*
anything validates it, and the `return` at line 217 exits before the `runtime`
group — which is where `NEXT_PUBLIC_ENVIRONMENT: oneOf(ENVIRONMENTS).required()`
lives. So the check that would catch the bad value is skipped by the branch that
the bad value selected. Any string outside the exact four lowercase names makes
`isDeployed` false, which the code treats as "this is a laptop".

**Evidence:** `packages/shared/utils/env-utils.js:196-217`:

```js
  const environment = env.NEXT_PUBLIC_ENVIRONMENT;
  const isDeployed = DEPLOYED_ENVIRONMENTS.includes(environment);

  if (SKIP_VALUES.includes(String(env[SKIP_FLAG]).toLowerCase())) {
    if (isDeployed) {
      ...
    } else {
      ...
      return;
    }
  }
```

Run against the real creative-portal schema with `SKIP_ENV_VALIDATION=1`,
`MFA_DISABLED=true` and `SERVICES_URI_REWRITE=internal.svc=https://attacker.test`:

```
env="production" -> THREW (2 problems)
env="Production" -> NO THROW (nothing validated; MFA off + rewrite accepted)
env="prod"       -> NO THROW (nothing validated; MFA off + rewrite accepted)
env=undefined    -> NO THROW (nothing validated; MFA off + rewrite accepted)
```

**Impact:** The comment at lines 201-204 states *"An env var must not be able to
switch them off."* A second env var switches them off completely. `MFA_DISABLED`
is read at `packages/shared/api/login/index.ts:21`
(`const MFAEnabled = process.env.MFA_DISABLED !== "true";`), so this is a live
authentication control, not only a routing one.

**Fix:** Validate `NEXT_PUBLIC_ENVIRONMENT` *before* the skip branch, and treat an
unrecognised value as deployed rather than as local:

```js
if (environment !== undefined && !ENVIRONMENTS.includes(environment)) {
  throw new Error(
    `[${name}] NEXT_PUBLIC_ENVIRONMENT must be one of: ` +
      `${ENVIRONMENTS.join(", ")}`
  );
}

// Unknown or absent => assume deployed. Failing towards "guarded"
// is the only safe default for a value that gates the guards.
const isDeployed = environment !== "local";
```

---

### 3. The look-alike host protection — the change's stated security property — is untested

**[File: packages/shared/api/services/rewriteServiceUri.test.ts:140-147]**

> **In plain terms:** The change promises that a web address merely *resembling*
> a real backend cannot impersonate it and collect the secret key we send along.
> We deliberately broke that protection and every single test still passed, so
> nothing would tell us if a future edit weakened it. The one test aimed at this
> checks the harmless direction and misses the dangerous one.

**Function/Class:** `rewriteServiceUri`

**Severity:** high

**Confidence:** high

**How to spot it:** Not user-reproducible — this is a test-coverage gap on a
security-relevant line. Change `rewriteServiceUri.ts:130` from
`candidate.fromHost === host` to `host.endsWith(candidate.fromHost)` and run
`yarn package:shared test rewriteServiceUri`: all 24 tests still pass.

**Problem:** The existing test uses `private.example.com.attacker.test`, where the
configured host is a **prefix** of the attacker host. A suffix-matching regression
rejects that too, so the test cannot detect it. The dangerous direction — an
attacker host that *ends with* the configured host, e.g.
`evil.private.example.com` — has no test.

**Evidence:** `packages/shared/api/services/rewriteServiceUri.test.ts:140-147`:

```ts
  it("is not fooled by a lookalike suffix host", () => {
    expect(
      rewriteServiceUri(
        "http://private.example.com.attacker.test/orders",
        REWRITES
      )
    ).toBe("http://private.example.com.attacker.test/orders");
  });
```

Mutation run (real, with a positive control proving the harness detects
regressions — changing `/[/?#\\]/` to `/[/?#]/` correctly killed 1 test):

| Mutation | Result |
|---|---|
| `candidate.fromHost === host` → `host.endsWith(...)` | **SURVIVED** — 24/24 pass |
| *positive control:* `/[/?#\\]/` → `/[/?#]/` | KILLED — 1 failed / 23 passed |

Under the surviving mutation, `rewriteServiceUri("http://evil.private.example.com/orders", REWRITES)`
returns `https://public.example.com/orders` — a rewrite that should not have fired.

**Impact:** `getAll.ts:59-63` sends the `api-key` header to the rewritten origin,
and `prepareServiceAxiosConfigs.ts` attaches each per-service key to every call
built from `service.uri`. A regression here silently redirects credential-bearing
traffic, and CI stays green. The code is correct today; the safety net is missing.

**Fix:** Add the suffix direction:

```ts
it("is not fooled by a host that ends with the configured host", () => {
  expect(
    rewriteServiceUri("http://evil.private.example.com/orders", REWRITES)
  ).toBe("http://evil.private.example.com/orders");
});
```

---

### 4. Eighteen of the nineteen customer-portal tests pass with validation entirely disabled

**[File: apps/customer-portal/env.test.ts:36-65]**

> **In plain terms:** We switched the whole validation feature off — made it
> incapable of reporting any problem — and this portal's tests still all reported
> success. They are written so that if nothing goes wrong, they check nothing and
> quietly pass. About a fifth of the change's advertised test count provides no
> protection.

**Function/Class:** `customerPortalEnv` canary suite

**Severity:** high

**Confidence:** high

**How to spot it:** Not user-reproducible — test-quality defect. Change
`env-utils.js:236` from `if (errors.length > 0)` to `if (false)` and run
`yarn app:customer-portal test env`: **19/19 pass**. The same mutation correctly
fails 1 creative-portal test.

**Problem:** Every assertion sits inside a `catch` block with no
`expect.assertions()` and no assertion that a throw occurred. If the code under
test stops throwing, the `catch` never runs, zero assertions execute, and vitest
reports a pass.

**Evidence:** `apps/customer-portal/env.test.ts:36-50`:

```ts
  keys.forEach((key) => {
    it(`${key} does not echo its value in the error`, () => {
      try {
        run({
          NEXT_PUBLIC_ENVIRONMENT: "production",
          [key]: CANARY
        });
      } catch (error) {
        expect((error as Error).message).not.toContain(CANARY);
      }
    });
  });
```

Same shape at `:52-65`. Only `:28-34` ("declares the keys the portal actually
reads") asserts outside a `try`, which is why 1 of 19 is sound. The creative
portal's equivalent avoids this by asserting `.toThrow(/is required/)` first — so
this is an inconsistency within the PR, not a missing idiom. The PR already uses
the right pattern elsewhere: `env-utils.test.ts:155` calls
`expect.unreachable("should have thrown")`.

**Impact:** 17 canary tests and the "exempts a developer machine" test are
vacuous. The "never echoes a secret" guarantee — the reason this file exists — is
not actually enforced for the customer portal. Compounding it, `apps/customer-portal/env.js`
contains **zero** `typeError` overrides (creative has 4) because it has no numeric
fields, so those 17 canaries guard nothing today even when they do run.

**Fix:** Assert the throw, then inspect it:

```ts
it(`${key} does not echo its value in the error`, () => {
  expect(() =>
    run({ NEXT_PUBLIC_ENVIRONMENT: "production", [key]: CANARY })
  ).toThrow();

  try {
    run({ NEXT_PUBLIC_ENVIRONMENT: "production", [key]: CANARY });
    expect.unreachable("should have thrown");
  } catch (error) {
    expect((error as Error).message).not.toContain(CANARY);
  }
});
```

---

### 5. A devtest server with MFA disabled cannot boot, and the escape hatch its own template documents is refused there

**[File: packages/shared/utils/env-utils.js:47 and :199-209]**

> **In plain terms:** Test servers commonly have two-factor login switched off so
> testers can log in quickly. After this change, such a server refuses to start
> at all. The setup file tells the operator there is an override flag that works
> everywhere except the two production-like environments — but the code refuses
> it on test servers too, so the documented recovery does not work. The only way
> out is editing the server's configuration by hand.

**Function/Class:** `validateEnvConfig`

**Severity:** high

**Confidence:** high — the behaviour is confirmed; whether devtest VMs currently
carry `MFA_DISABLED=true` is an open question (see Open Questions)

**Steps to reproduce:**

1. On a deployed devtest VM, `.env` contains `NEXT_PUBLIC_ENVIRONMENT=devtest`
   and `MFA_DISABLED=true`.
2. Deploy this branch and start the server.
3. **Expected:** the server starts, as it does today.
4. **Actual:** validation fails and `instrumentation.ts:31` calls `process.exit(1)`.
5. The operator reads `.env.example:48-51`, which says the override is only
   ignored on stage and production, and sets `SKIP_ENV_VALIDATION=1`.
6. **Expected:** the server starts.
7. **Actual:** the flag is refused and the server still exits.

**Problem:** `DEPLOYED_ENVIRONMENTS` includes `devtest`, so both the `MFA_DISABLED`
guard and the `SKIP_ENV_VALIDATION` refusal apply there — but both `.env.example`
files and both `env.js` files document the scope as "stage or production".

**Evidence:** `packages/shared/utils/env-utils.js:47`:

```js
const DEPLOYED_ENVIRONMENTS = ["devtest", "stage", "production"];
```

`apps/creative-portal/.env.example:48-51` (customer `:47-50` identical):

```
# Escape hatch for local troubleshooting only: set to 1 to skip
# environment validation entirely. Ignored, and logged as ignored,
# when NEXT_PUBLIC_ENVIRONMENT is stage or production.
SKIP_ENV_VALIDATION=
```

Run against the real schema:

```
=== A deployed devtest box carrying MFA_DISABLED=true ===
  BOOT BLOCKED -> process.exit(1):
     - MFA_DISABLED must not disable MFA in a deployed environment [devtest]

=== operator applies the documented escape hatch ===
[creative-portal] SKIP_ENV_VALIDATION is ignored when NEXT_PUBLIC_ENVIRONMENT is "devtest".
  STILL BLOCKED - escape hatch refused:
     - MFA_DISABLED must not disable MFA in a deployed environment [devtest]
```

**Impact:** A devtest deploy that fails this way cannot be recovered by the
documented route. `MFA_DISABLED` was previously unconstrained, and the template
at `:44-45` describes it as a normal dev toggle, so a devtest box carrying it is
plausible rather than exotic.

**Fix:** Decide which is intended and make code and docs agree. If devtest should
be guarded (which the ticket argues for), correct both `.env.example` files and
the `deployed:` comments in both `env.js` files, and confirm the devtest VMs
before merging. If devtest should keep the escape hatch, restrict the refusal to
`["stage", "production"]`.

---

### 6. A whitespace-only value in an optional variable blocks boot

**[File: packages/shared/utils/env-utils.js:149-159]**

> **In plain terms:** An optional setting left as a single space instead of truly
> empty now stops the server from starting. A stray space is invisible in most
> configuration tools and neither container environment variables nor deployment
> config files trim them, so this is an easy way to take a server down over a
> setting that does not matter.

**Function/Class:** `blankAsMissing`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. In a portal `.env`, set an optional analytics key to a single space, e.g.
   `NEXT_PUBLIC_HOTJAR_SITE_ID= ` (note the trailing space).
2. Start the server.
3. **Expected:** the field is optional, so it is ignored.
4. **Actual:** validation fails with
   `NEXT_PUBLIC_HOTJAR_SITE_ID must be a number [runtime]` and the process exits.

**Problem:** Only the exactly-empty string is normalised to "missing". A
whitespace-only value is treated as a real value and validated against the
field's type rules.

**Evidence:** `packages/shared/utils/env-utils.js:153-157`:

```js
      schema.transform((value, originalValue) =>
        originalValue === "" || originalValue === null
          ? undefined
          : value
      )
```

Run against the real creative-portal schema:

```
  HOTJAR_SITE_ID/SENTRY_AUTH_TOKEN="" -> NO THROW
  HOTJAR_SITE_ID/SENTRY_AUTH_TOKEN=" " -> THREW:
     - SENTRY_AUTH_TOKEN must be at least 16 characters [runtime]
     - NEXT_PUBLIC_HOTJAR_SITE_ID must be a number [runtime]
```

**Impact:** Because `instrumentation.ts:31` exits the process, a stray space in a
field explicitly declared `.notRequired()` is a boot failure. Also note
`MASTER_REFERENCE_VERSION="   "` passes `.required()` — the same gap in the other
direction.

**Fix:**

```js
      schema.transform((value, originalValue) =>
        originalValue === null ||
        (typeof originalValue === "string" &&
          originalValue.trim() === "")
          ? undefined
          : value
      )
```

---

### 7. The boot-time rewrite check accepts entries the parser then silently discards

**[File: packages/shared/utils/env-utils.js:100-141]**

> **In plain terms:** A setting was added specifically so that a typo in the
> local backend redirect is reported at start-up instead of silently doing
> nothing. It only inspects half of each entry, so the most likely typos pass the
> check and are then thrown away anyway. The developer gets a clean start-up and
> a redirect that never happens — exactly the confusion the check was meant to
> remove.

**Function/Class:** `serviceUriRewrite`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Set `SERVICES_URI_REWRITE=a.test=https://b.test, c.test:x=https://d.test` in a
   portal `.env` (the second entry has a malformed port).
2. `yarn dev:creative-portal`
3. **Expected:** start-up reports the malformed entry.
4. **Actual:** start-up is clean, and only the first of the two redirects is
   active. The second is discarded with no message.

**Problem:** The validator checks `separator > 0` and then validates only the
right-hand side. The left-hand side is never parsed, while
`parseServiceUriRewrites` does parse it and drops the entry when it fails.

**Evidence:** `packages/shared/utils/env-utils.js:114-121`:

```js
        .every((entry) => {
          const separator = entry.indexOf("=");

          if (separator <= 0) {
            return false;
          }

          const target = entry.slice(separator + 1).trim();
```

versus `packages/shared/api/services/rewriteServiceUri.ts:95-102`:

```ts
      const fromHost = parseHost(
        entry.slice(0, separatorIndex).trim()
      );
      ...
      return fromHost && toOrigin ? { fromHost, toOrigin } : null;
```

Running both shipped code paths over the same inputs:

```
"a.test:notaport=https://b.test"                 validator: true | entries: 1 | parser produced: 0
"a b.test=https://b.test"                        validator: true | entries: 1 | parser produced: 0
"http://=https://b.test"                         validator: true | entries: 1 | parser produced: 0
"a.test=https://b.test, c.test:x=https://d.test" validator: true | entries: 2 | parser produced: 1
```

**Impact:** The validator's own docstring (`env-utils.js:93-98`) states it exists
so a malformed value does not "degrade to 'no rewrite' with no signal at all".
For left-hand-side typos it does exactly that. No input was found in the opposite
direction.

**Fix:** Reuse the parser rather than maintaining a second implementation:

```js
const serviceUriRewrite = () =>
  Yup.string().test(
    "uri-rewrite-format",
    "must be comma separated fromHost=toOrigin pairs, each " +
      "target an http or https origin",
    (value) => {
      if (value === undefined) {
        return true;
      }

      const entries = value
        .split(",")
        .map((entry) => entry.trim())
        .filter(Boolean);

      return (
        parseServiceUriRewrites(value).length === entries.length
      );
    }
  );
```

---

### 8. A rewrite entry naming a default port matches a different set of addresses than written

**[File: packages/shared/api/services/rewriteServiceUri.ts:12-13]**

> **In plain terms:** If a developer writes the redirect using an address that
> includes port 443 or port 80, the redirect quietly applies to a *different* set
> of backend addresses than the one written — and does not apply to the one that
> was actually named. Since the redirect carries a secret key along with it,
> traffic can be sent somewhere the developer did not ask for.

**Function/Class:** `parseHost` / `withScheme`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Set `SERVICES_URI_REWRITE=a.test:443=https://new.test`.
2. Have the Directory Service return `http://a.test:443/orders` (the address
   named) and `https://a.test/orders` (one that was not).
3. **Expected:** `http://a.test:443/orders` is redirected; `https://a.test/orders`
   is left alone.
4. **Actual:** the opposite. `https://a.test/orders` is redirected to
   `https://new.test/orders`; `http://a.test:443/orders` is untouched.

**Problem:** The configuration side is always parsed as `https`
(`withScheme` prepends `https://` to a scheme-less value), while the match side
uses the URI's real scheme. `URL.host` omits the port only when it is the default
*for that scheme*, so the two sides disagree for ports 443 and 80.

**Evidence:** `packages/shared/api/services/rewriteServiceUri.ts:12-13`:

```ts
const withScheme = (value: string) =>
  value.includes("://") ? value : `https://${value}`;
```

Verified:

```
config "a.test:443" -> parseHost = "a.test"
config "a.test:80"  -> parseHost = "a.test:80"
uri http://a.test:443/p  -> host = "a.test:443"
uri https://a.test/p     -> host = "a.test"
uri http://a.test/p      -> host = "a.test"
```

The port tests at `rewriteServiceUri.test.ts:170-188` use `8080` only, so the
default-port cases are uncovered.

**Impact:** A rewrite that attaches `api-key` headers is applied to a host the
operator did not name, and skips the one they did. Bounded to local development,
since a non-empty `SERVICES_URI_REWRITE` is rejected in deployed environments —
but that rejection is itself bypassable (Issues 2 and 9).

**Fix:** Normalise both sides through the same scheme — parse the left-hand side
using the scheme of the URI being matched, or require an explicit scheme on the
left-hand side and reject a bare `host:port`.

---

### 9. `NEXT_PUBLIC_ENVIRONMENT=local` on a deployed server disables every guard, silently

**[File: packages/shared/utils/env-utils.js:196-197 and :232]**

> **In plain terms:** All the new protections are switched on by a single line in
> the server's own configuration file that says which environment it is. If that
> line says "local" — which is exactly what it would say if someone copied a
> developer's configuration onto a server — every protection turns off, with no
> warning message at all. The error reporting that would have surfaced the
> mistake is switched off by the same value.

**Function/Class:** `validateEnvConfig`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. On a production VM, set `NEXT_PUBLIC_ENVIRONMENT=local` (e.g. by copying a
   working laptop `.env`), keeping `SERVICES_URI_REWRITE` and `MFA_DISABLED=true`
   alongside it.
2. Boot the server.
3. **Expected:** the deployed guards reject both values.
4. **Actual:** the server boots normally. No warning is logged.

**Problem:** `local` is a valid value in `ENVIRONMENTS`, so the runtime group
accepts it; `isDeployed` is then false and line 232 skips the `deployed` group
entirely. Unlike the `SKIP_ENV_VALIDATION` path, this emits no console output at all.

**Evidence:** `packages/shared/utils/env-utils.js:196-197` and `:232`:

```js
  const environment = env.NEXT_PUBLIC_ENVIRONMENT;
  const isDeployed = DEPLOYED_ENVIRONMENTS.includes(environment);
...
  if (isDeployed && groups.includes("deployed")) {
```

Compounding: `getSentryDsn("local", portal)` returns `""`
(`packages/shared/config/sentry/keys.ts:33-36`), and with the new `enabled: !!dsn`
(`baseInit.ts:104`) Sentry is off — so the misconfiguration that disables the
guards also disables the reporting that would reveal it.

**Impact:** The guard catches only the narrower accident where someone copies
`SERVICES_URI_REWRITE` but *not* `NEXT_PUBLIC_ENVIRONMENT`. The comment at
`env-utils.js:201-204` claims more than the code delivers.

**Fix:** Derive deployed-ness from a signal that does not live in the same file as
the values being guarded, or at minimum fail closed when
`NEXT_PUBLIC_ENVIRONMENT === "local"` and `NODE_ENV === "production"`. A
deploy-time assertion that `local` never ships would also close it.

---

### 10. In production, a module-loading failure in the boot hook is swallowed and the server starts unvalidated

**[File: apps/creative-portal/instrumentation.ts:16, apps/customer-portal/instrumentation.ts:16]**

> **In plain terms:** The new start-up safety check can vanish without anyone
> noticing. If the packaging step fails to include one of the files the check
> depends on, the framework quietly ignores the failure and starts the server
> with no checks at all. Nothing is logged, and the server looks completely
> healthy.

**Function/Class:** `register`

**Severity:** medium

**Confidence:** high

**How to spot it:** Not routinely user-reproducible. Inspect a built
`.next/standalone` tree and confirm `yup`, `apps/*/env.js` and
`packages/shared/utils/env-utils.js` are all present. If any is missing, the
server boots with validation silently absent.

**Problem:** `await import("./env")` sits outside the `try`, and more importantly
Next 14.1.1 treats a `MODULE_NOT_FOUND` from the instrumentation hook as
"no instrumentation" rather than as an error — but only in production, which is
where it matters. `env.js` reaches `require("yup")` through
`packages/shared/utils/env-utils.js:2`, so a tracing miss for yup produces exactly
that error code.

**Evidence:** `apps/creative-portal/instrumentation.ts:16-19`:

```ts
    const { validateCreativePortalEnv } = await import("./env");

    try {
      validateCreativePortalEnv();
```

`node_modules/next/dist/server/next-server.js:436-441`:

```js
            } catch (err) {
                if (err.code !== "MODULE_NOT_FOUND") {
                    err.message = `An error occurred while loading instrumentation hook: ${err.message}`;
                    throw err;
                }
            }
```

The dev server rethrows unconditionally
(`next/dist/server/dev/next-dev-server.js:426-428`), so this asymmetry hides in
production only.

**Impact:** The PR's central "fail closed at boot" property can disappear with no
signal. Note this is *narrower* than a general fail-open: a non-`MODULE_NOT_FOUND`
throw from `register()` does kill the process (see Issue 18).

**Fix:** Move the import inside the `try`, and make the absence of validation
detectable — e.g. have `register()` log a single confirmation line on success, so
its absence is visible in boot logs.

---

### 11. Three option-handling branches that gate the whole feature have no test coverage

**[File: packages/shared/utils/env-utils.js:199, :221, :167-168]**

> **In plain terms:** The parts of the new code that decide *which* checks run,
> and *whether* they can be skipped, have no tests at all. We broke each of them
> deliberately and the suite stayed green — including a change the code's own
> comment specifically warns against.

**Function/Class:** `validateEnvConfig` / `validateGroup`

**Severity:** medium

**Confidence:** high

**How to spot it:** Not user-reproducible — coverage gap. `grep -rn "groups"`
across all three new test files returns no occurrences.

**Problem:** Real mutation results (positive control confirmed the harness works):

| Mutation | File:line | Result |
|---|---|---|
| `options.groups \|\| [...]` → always `[...]` | `env-utils.js:221` | **SURVIVED** 28/28 |
| allow-list → `if (env[SKIP_FLAG])` | `env-utils.js:199` | **SURVIVED** 28/28 |
| `if (isDeployed)` → `if (false)` | `env-utils.js:200` | KILLED |
| `isDeployed = ...` → `= false` | `env-utils.js:197` | KILLED |

**Evidence:** The `groups` narrowing is the exact path both `next.config.js:11`
call sites use, and the 7-line rationale at `env-utils.js:182-188` is entirely
unverified. The `SKIP_VALUES` allow-list carries this comment at `:25-28`:

```js
// An explicit allow-list, not a truthiness check: "false" and "0"
// are truthy strings, so `if (env[SKIP_FLAG])` would let someone
// who wrote SKIP_ENV_VALIDATION=false to *enforce* validation
// silently switch it off instead.
```

Applying precisely that regression passes all 28 tests. Only `"1"` and `""` are
tested, and `""` is falsy so it cannot distinguish the two implementations.
Separately, the `[error]` fallback at `:168` is unreachable — yup 0.32.11 always
populates `.inner` under `abortEarly: false`.

**Impact:** The build-time gate could silently validate nothing (or validate
runtime secrets and fail every CI build) with no test failing, and the documented
allow-list behaviour could regress unnoticed.

**Fix:** Add a test per branch: `{ groups: ["build"] }` validates only the build
group; `SKIP_ENV_VALIDATION=false` and `=0` do **not** skip; `=TRUE` and `=yes` do.

---

### 12. Unsourced 16-character minimums on API keys are now boot-fatal

**[File: apps/creative-portal/env.js:26-28, apps/customer-portal/env.js:24-26 and :38-40]**

> **In plain terms:** The change invents a minimum length for several secret keys
> and refuses to start the server if a real key is shorter. Nothing in the code
> actually needs that length — it is a guess. If any real key is shorter, every
> server refuses to start on the first deploy after this merges.

**Function/Class:** `creativePortalEnv.runtime` / `customerPortalEnv.runtime`

**Severity:** medium

**Confidence:** high (the constraint is confirmed; whether real keys are shorter
is an open question)

**How to spot it:** Compare the live `SERVICES_API_KEY`, `CLIENT_SERVICES_API_KEY`
and `GOOGLE_CLIENT_SECRET` values against 16 characters before merging.

**Problem:** The only consumer sends the key as a header without inspecting it
(`getAll.ts:59-63`). The previous floor was 1 character
(`runtimeParams = { min: 1, max: 200 }`).

**Evidence:** `apps/creative-portal/env.js:26-28`:

```js
    SERVICES_API_KEY: Yup.string()
      .min(16, "must be at least 16 characters")
      .required("is required"),
```

For contrast, `SECRET_COOKIE_PASSWORD: .min(32)` **is** justified — `iron-session`
enforces it at `packages/shared/lib/session.ts:5`. The 16s have no such basis.

**Impact:** A key of 15 characters or fewer becomes a fleet-wide `process.exit(1)`.
The check adds no security — it does not make a short key safe or an attacker's
job harder — but it can take the estate down.

**Fix:** Drop the `.min(16)` on the API keys and Google secret, or confirm the real
values first. Keep `.required()`, which is genuinely correct.

---

### 13. Feature-scoped Google credentials now stop the whole customer portal from booting

**[File: apps/customer-portal/env.js:37-40 and :55-58]**

> **In plain terms:** Five settings are only needed by the "attach from Google
> Drive" feature. Previously, a missing one meant that one button did not work.
> Now it means the entire customer portal refuses to start.

**Function/Class:** `customerPortalEnv.runtime`

**Severity:** medium

**Confidence:** high

**How to spot it:** Not user-reproducible today — a change in failure mode.
Remove any Google key from a customer-portal `.env` and start the server.

**Problem:** All five are genuinely read, but only on the Drive path
(`apps/customer-portal/utils/oAuth2Client.ts:5-9`;
`components/pages/create-orders/index.tsx:133-134`). Marking them `.required()`
escalates a feature outage into a service outage.
`NEXT_PUBLIC_GOOGLE_AUTH_CALLBACK` is additionally tightened to `httpUrl()`,
which rejects any relative value the old validator accepted.

**Evidence:** `apps/customer-portal/env.js:55-58`:

```js
    NEXT_PUBLIC_GOOGLE_API_KEY: Yup.string().required("is required"),
    NEXT_PUBLIC_GOOGLE_APP_ID: Yup.string().required("is required"),
    NEXT_PUBLIC_GOOGLE_AUTH_CALLBACK:
      httpUrl().required("is required"),
```

**Impact:** Larger blast radius than the feature warrants, and a new absolute-URL
constraint on a value nobody has checked.

**Fix:** Consider `.notRequired()` for the Drive-only keys, matching the treatment
Hotjar and Amplitude already get on the creative side (whose guard
`if (hjid && hjsv)` the schema comment explicitly cites as the reason).

---

### 14. The production change in `getAll.ts` has no test

**[File: packages/shared/api/services/getAll.ts:38-46 and :71]**

> **In plain terms:** The new redirect helpers are well tested in isolation, but
> the place where they are actually wired into the app is not tested at all.
> Deleting the line that applies the redirect to each backend address would break
> nothing in the suite.

**Function/Class:** `fetchServices`

**Severity:** medium

**Confidence:** high

**How to spot it:** `grep -rln "fetchServices\|getAllServices" --include=*.test.ts`
returns no matches.

**Problem:** Nothing verifies that `SERVICES_URI_REWRITE` is threaded from
`config/envs`, nor that per-service `item.uri` is rewritten.

**Evidence:** `packages/shared/api/services/getAll.ts:69-73`:

```ts
  const servicesWithVersion = data.map((item) => ({
    ...item,
    uri: rewriteServiceUri(item.uri, uriRewrites),
    version: apiVersion ?? ""
  }));
```

**Impact:** The 25 unit tests cover the helpers thoroughly; the integration point
they exist for is unguarded.

**Fix:** One test mocking `fetchJson` and `config/envs`, asserting both the
directory URL and each returned `service.uri` come back rewritten.

---

### 15. Control characters between the scheme and the address corrupt the rewritten path

**[File: packages/shared/api/services/rewriteServiceUri.ts:58-65]**

> **In plain terms:** If a backend address contains an invisible control
> character in an unusual spot, the redirect sends the request to the right
> server but the wrong page, producing a "not found" error that would be hard to
> diagnose. It cannot send the request to the wrong server.

**Function/Class:** `afterAuthority`

**Severity:** low

**Confidence:** high

**How to spot it:** Not user-reproducible without a Directory Service returning
such a URI. `rewriteServiceUri("https:/\r/directory.internal/api/orders", rewrites)`
returns `https://localhost:8443/directory.internal/api/orders`.

**Problem:** The loop skips only `/` and `\`. WHATWG additionally *removes* tab,
LF and CR anywhere in the input before parsing, so one of those characters in the
slash run stops the loop early while the parsed host is unaffected. The comment at
`:6-9` reasons about `\` but not about the characters WHATWG deletes outright.

**Evidence:** `packages/shared/api/services/rewriteServiceUri.ts:60-65`:

```ts
  while (
    authorityStart < uri.length &&
    (uri[authorityStart] === "/" || uri[authorityStart] === "\\")
  ) {
    authorityStart += 1;
  }
```

Verified:

```
"https:/\r/directory.internal/api/orders?version=25.3"
   -> WHATWG host = "directory.internal"  path+search = "/api/orders?version=25.3"
```

so the string surgery yields `/directory.internal/api/orders?version=25.3`.

**Impact:** Wrong resource, not wrong origin — `afterAuthority` can only return a
string starting with `/`, `?`, `#` or `\`, so the result always re-parses onto
`toOrigin`. Bounded to local development.

**Fix:** Strip `\t`, `\n` and `\r` from `uri` before the string surgery, matching
what WHATWG does before parsing.

---

### 16. Documentation contradicts the code about which environments are guarded

**[File: apps/creative-portal/env.js:83-84, apps/customer-portal/env.js:70-71, both `.env.example` files]**

> **In plain terms:** The comments next to the new safety checks say they apply to
> two environments; the code applies them to three. Since the whole point of the
> change was to extend the checks to that third environment, the comments describe
> the problem rather than the fix.

**Function/Class:** `creativePortalEnv.deployed` / `customerPortalEnv.deployed`

**Severity:** low

**Confidence:** high

**How to spot it:** Compare `DEPLOYED_ENVIRONMENTS` at `env-utils.js:47` with the
comments cited below.

**Problem:** Stale wording from before the `local` split.

**Evidence:** `apps/creative-portal/env.js:83-84`:

```js
  // Applied when NEXT_PUBLIC_ENVIRONMENT names a deployed
  // environment (stage or production), not just production.
```

`DEPLOYED_ENVIRONMENTS` is `["devtest", "stage", "production"]`. The same drift is
in `apps/creative-portal/.env.example:48-51` and
`apps/customer-portal/.env.example:47-50`.

**Impact:** Directly causes the failed recovery in Issue 5.

**Fix:** Change "(stage or production)" to "(devtest, stage or production)" in all
four places.

---

### 17. Two `.toLowerCase()` calls are dead, and the two tests naming them are vacuous

**[File: packages/shared/api/services/rewriteServiceUri.ts:17 and :124]**

> **In plain terms:** Two lines guard against differences in capitalisation, but
> the tool they use already handles that, so the lines do nothing. Two tests claim
> to check those lines but are really checking the tool.

**Function/Class:** `parseHost` / `rewriteServiceUri`

**Severity:** low

**Confidence:** high

**How to spot it:** Delete either `.toLowerCase()` and run
`yarn package:shared test rewriteServiceUri` — 24/24 still pass.

**Problem:** WHATWG `new URL()` already lowercases the host for special schemes,
and `withScheme` forces `https://` on scheme-less input.

**Evidence:** Mutation results — dropping `.toLowerCase()` at `:124` **SURVIVED**
24/24; dropping it at `:17` **SURVIVED** 24/24. Tests
`rewriteServiceUri.test.ts:73-79` ("lowercases the matched host") and `:128-132`
("matches a host with different casing") exercise the URL parser, not the lines
they name.

**Impact:** Harmless but misleading — the tests imply coverage that does not exist.

**Fix:** Keep the calls as defensive belt-and-braces if preferred, but retitle the
tests to describe what they actually assert.

---

### 18. The comment justifying `process.exit(1)` is wrong for Next 14.1.1

**[File: apps/creative-portal/instrumentation.ts:21-25, apps/customer-portal/instrumentation.ts:21-25]**

> **In plain terms:** A comment explains why a piece of defensive code is needed,
> using a description of the framework's behaviour that is not accurate for the
> version in use. The code is fine; a future reader trusting the comment would
> draw a wrong conclusion.

**Function/Class:** `register`

**Severity:** low

**Confidence:** high

**How to spot it:** Read `next/dist/server/lib/start-server.js:264` and `:272-275`.

**Problem:** The comment states Next keeps the process alive and still logs
"Ready". In 14.1.1 the opposite happens.

**Evidence:** `apps/creative-portal/instrumentation.ts:21-25`:

```ts
      // Next catches a throw from this hook and keeps the process
      // alive, still logging "Ready" while answering every request
      // with a 500.
```

`next/dist/server/lib/start-server.js` places `_log.event('Ready in ...')` at
line 264 inside the same `try` as `await getRequestHandlers(...)` at line 247, and
the catch at `:272-275` is annotated *"fatal error if we can't setup"* followed by
`process.exit(1)`. So a throw exits the process **before** "Ready" is logged.

**Impact:** None functionally — the explicit `process.exit(1)` remains worthwhile
defence-in-depth (and produces a far better error message than a raw stack). Only
the rationale is wrong.

**Fix:** Reword to describe the real benefit: a clean, single-line message instead
of a wrapped stack trace, and independence from Next's internal error handling.

---

### 19. One missing variable is reported as two problems

**[File: apps/creative-portal/env.js:51 and :81]**

> **In plain terms:** When one setting is missing, the error message lists it
> twice, which makes the list of problems look worse than it is.

**Function/Class:** `creativePortalEnv` / `customerPortalEnv`

**Severity:** low

**Confidence:** high

**How to spot it:** Boot with `NEXT_PUBLIC_APP_VERSION` unset.

**Problem:** The key is declared in both the `runtime` and `build` groups, and the
boot-time call validates both.

**Evidence:** Observed output:

```
[creative-portal] environment validation failed with 2 problem(s):
  - NEXT_PUBLIC_APP_VERSION is required [runtime]
  - NEXT_PUBLIC_APP_VERSION is required [build]
```

**Impact:** Cosmetic; slightly undermines the "reports every problem at once"
framing since the count is inflated.

**Fix:** Dedupe by `path` before formatting, or accept it as intentional given the
groups are validated independently.

---

### 20. Three claims in the PR description and ticket do not match the base branch

**[File: PR description / PP-2119 ticket]**

> **In plain terms:** The write-up describes some clean-up that the code does not
> actually show. The work itself is fine — the record of it is inaccurate, which
> matters because the ticket is what the team will read later.

**Function/Class:** n/a — documentation accuracy

**Severity:** low

**Confidence:** high

**How to spot it:** `git grep "GOOGLE" e5e016eec -- apps/creative-portal` → 0 hits.

**Problem:** Three claims do not hold against `e5e016eec`:

1. *"moved 5 Google keys off creative-portal (no Google Drive route there)"* —
   creative-portal never had them. Its pre-PR `env.js` declares no Google key and
   its pre-PR `.env.example` is 14 lines with none. Nothing was moved; the keys
   were and remain customer-portal-only.
2. *"removed 5 dead keys"* is 4. `NEXT_PUBLIC_WYSIWYG_LOGS` never existed anywhere
   in the repo, including `packages/wysiwyg`.
3. *"Added the 2 keys customer-portal genuinely needed but lacked"* — the schema
   gains six (`SERVICES_URI_REWRITE`, `MFA_DISABLED`, `SENTRY_ORG`,
   `SENTRY_PROJECT`, `SENTRY_AUTH_TOKEN`, `NODE_ENV`). Two is right only for
   `.env.example`.

**Evidence:** Pre-PR `apps/creative-portal/env.js` runtime keys:
`SECRET_COOKIE_PASSWORD, SERVICES_API_URL, SERVICES_API_KEY, MASTER_REFERENCE_VERSION,
LOGTAIL_SOURCE_TOKEN, TIPTAP_PRO_TOKEN, TIPTAP_CLOUD_APP_ID, TIPTAP_CLOUD_APP_SECRET,
NEXT_PUBLIC_*`. No Google key present.

**Impact:** The ticket is the durable record of an audit whose whole value is
trustworthiness. Overstated counts undercut that.

**Fix:** Correct the three claims in the PR body and the ticket.

---

### 21. Two loose ends the audit missed

**[File: apps/creative-portal/setup/api/utils/get-api-key.ts:1, packages/shared/scripts/nextConfig.js:9]**

> **In plain terms:** The clean-up was described as covering every setting. Two
> were not covered: one leftover reference to a setting that no longer exists
> anywhere, and one logging service whose required setting is now documented
> nowhere.

**Function/Class:** n/a

**Severity:** low

**Confidence:** high

**How to spot it:** `grep -rn "REACT_APP_API_KEY\|LOGTAIL"` across `apps/` and
`packages/`.

**Problem:** Two gaps:

1. `apps/creative-portal/setup/api/utils/get-api-key.ts:1` reads
   `process.env.REACT_APP_API_KEY` — in no schema and no template. It has no
   importers, so it should be deleted rather than added.
2. `LOGTAIL_SOURCE_TOKEN` was removed from both schemas, but
   `packages/shared/scripts/nextConfig.js:9` still wires `withLogtail` into the
   build (`:61`). The "differently named var" the claim points to,
   `SENTRY_LOGTAIL_SOURCE_TOKEN` (`packages/shared/scripts/bootstrap.mjs:77`), is
   written but never read. Neither template mentions any Logtail variable now.

**Evidence:** `apps/creative-portal/setup/api/utils/get-api-key.ts:1`:

```ts
export const API_KEY = process.env.REACT_APP_API_KEY;
```

**Impact:** Minor. The Logtail one means the env contract for log shipping is
undocumented in both templates.

**Fix:** Delete `get-api-key.ts`. Establish what `@logtail/next` reads and either
document it or remove `withLogtail`.

---

### 22. Stale comment and a latent trap in the shared yup mock

**[File: packages/shared/__mocks__/yup.ts:1 and :16-17]**

> **In plain terms:** A note in the test setup says a library is not installed;
> this change installs it. A future tidy-up that trusts the note could break the
> new validation tests in a confusing way.

**Function/Class:** yup manual mock

**Severity:** low

**Confidence:** high

**How to spot it:** Read the comment, then `packages/shared/package.json:70`.

**Problem:** The comment says *"Manual mock for yup (not installed in shared
package)"*, which the PR makes false. `packages/shared/vitest.config.ts:37` aliases
`yup` to that stub, which exports only `boolean` and `mixed`. `env-utils.js:2`
survives only because it uses `require("yup")` — native resolution, alias-exempt.
Converting it to an ESM `import` would silently hand it a stub with no `object`,
`string` or `number`.

**Evidence:** `packages/shared/__mocks__/yup.ts:1`:

```ts
// Manual mock for yup (not installed in shared package)
```

**Impact:** No current defect. It is a tripwire for the next person who touches
either file.

**Fix:** Update the comment, and add a note in `env-utils.js` explaining that the
`require` is load-bearing for the vitest alias.

---

## Open Questions

These could not be confirmed from the repository and are questions, not defects.
The first two gate the merge.

- **What is `NEXT_PUBLIC_ENVIRONMENT` set to on each devtest / stage / production
  VM?** It appears nowhere in the repo — `grep -rn "NEXT_PUBLIC_ENVIRONMENT" .github/`
  returns zero hits — so it lives only on the VMs. Any value outside the exact
  lowercase set `local | devtest | stage | production` now exits the process at
  boot, where today it only leaves Sentry without a DSN. Note
  `.github/workflows/deploy-creative-portal-devtest.yml:66` passes
  `ENVIRONMENT: "development"` to the deploy workflow — a different variable, but
  it shows `development` is the pipeline's vocabulary, and that string is not in
  `ENVIRONMENTS`. — `packages/shared/utils/env-utils.js:31`
- **Does `Proofed/B2BDeployment/.github/workflows/build-upload.yml@2.11.0` export
  `NEXT_PUBLIC_APP_VERSION` into the `next build` step?** Nothing in this repo
  sets it, and it is now a hard build gate with no bypass (`SKIP_ENV_VALIDATION`
  is deliberately inert for stage/production). That repo is not readable from
  here. Circumstantial evidence that it *is* set: `nextConfig.js:88-90` feeds it
  to the Sentry release name, and working client-side Sentry in devtest implies
  it is present at build. That is inference, not proof. — `apps/creative-portal/next.config.js:11`
- **Are the real `SERVICES_API_KEY`, `CLIENT_SERVICES_API_KEY` and
  `GOOGLE_CLIENT_SECRET` values all at least 16 characters, and is
  `NEXT_PUBLIC_GOOGLE_AUTH_CALLBACK` an absolute http(s) URL?** Any "no" is a
  fleet-wide boot failure on the first deploy. — `apps/creative-portal/env.js:26-28`
- **Do deployed devtest boxes currently run with `MFA_DISABLED=true`?** Determines
  whether Issue 5 fires on the next devtest deploy or stays theoretical. — `apps/customer-portal/env.js:80-83`
- **Is `SERVICES_URI_REWRITE` set on any deployed box today?** If so, that server
  will refuse to boot after this merges. — `apps/creative-portal/env.js:86-92`
- **Is there a supervisor that restarts the `StandaloneWebserver` task after
  `process.exit(1)`, and does the deploy gate on health rather than on the task
  starting?** If the task simply stops, a bad env becomes a silent, unalerted
  outage — made worse because `process.exit(1)` runs *before*
  `sentry.server.config` is imported (`instrumentation.ts:31-34`), so the one
  failure most worth paging on never reaches Sentry.
- **Does `npmrc-replace-env` load a root `.env`?** `README.md:54` and the new root
  `.env.example` both now instruct developers to put `TIPTAP_PRO_TOKEN` there.
  `package.json:37` runs it via `npx` with no dotenv preload. If it reads only
  `process.env`, the newly documented setup path produces an `.npmrc` containing
  the literal string `TIPTAP_PRO_TOKEN` and `yarn install` fails. — `README.md:53-54`
- **Should a URI carrying userinfo keep its credentials?**
  `http://user:pass@private.example.com/x` is rewritten to
  `https://public.example.com/x`, dropping them. Deliberate stripping or
  unconsidered? — `packages/shared/api/services/rewriteServiceUri.ts:133`

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

**The validation suite was not run.** The PR touches `packages/shared`, so a
faithful run means the full unscoped monorepo suite, which needs a complete
`yarn install` in a fresh worktree (`TIPTAP_PRO_TOKEN` was not available in the
review shell) and would collide with the known local full-suite hang.

**Separately: GitHub reports zero check runs on this PR.** The deploy workflows
trigger only on `develop` pushes and `oms-*` / `cp-*` tags, so nothing validates
a PR before merge. Whatever confidence exists comes from the author's local run
(reported as 2342 passed / 4 skipped / 1 pre-existing locale-dependent failure)
and from this review. **Re-run the suite before merging.**

Targeted verification that *was* performed, executing the real shipped schemas
and helpers against real yup 0.32.11 and the Node WHATWG URL parser:

- Fresh-template boot failure — reproduced (Issue 1)
- `SKIP_ENV_VALIDATION` + misspelled environment — reproduced (Issue 2)
- Devtest `MFA_DISABLED` boot block and refused escape hatch — reproduced (Issue 5)
- Whitespace-only optional value — reproduced (Issue 6)
- Validator/parser divergence — reproduced across 4 inputs (Issue 7)
- Default-port host asymmetry — reproduced (Issue 8)
- Control-character path corruption — reproduced (Issue 15)
- 9 mutations run against the test suite with a passing positive control (Issues 3, 4, 11, 17)
- Next 14.1.1 instrumentation error handling read in `node_modules` (Issues 10, 18)

---

## Tests

- ✅ 89 tests added where the touched modules previously had none — a real and
  substantial improvement over `develop`, where this code had zero coverage.
- ✅ `apps/*/env.test.ts` **are** collected: both `vitest.config.ts` files use
  `include: ["**/*.test.{ts,tsx}"]`, and the app-root path is not excluded.
- ✅ `rewriteServiceUri.test.ts` is genuinely thorough — 25 tests covering
  backslash authorities, scheme-relative URIs, casing, idempotence and third-party
  hosts.
- ✅ "Never echoes a value" is properly guarded for the creative portal, and the
  claim itself verified: 0 echoes across both schemas × 19 canary values.
- ❌ The look-alike suffix security property is untested (Issue 3).
- ❌ 18 of 19 customer-portal tests pass with the feature disabled (Issue 4).
- ❌ The `groups` selection and the `SKIP_VALUES` allow-list are untested (Issue 11).
- ❌ `getAll.ts` — the actual production integration — has no test (Issue 14).
- ❌ No test for `instrumentation.ts`, so the `process.exit(1)` behaviour the
  design rests on is unverified.
- ⚠️ Test-count inflation: 60 authored `it()` blocks expand to ~91 runtime tests;
  33 are per-key canary repetitions asserting one identical property.

### Suggested manual QA script

1. **(Issue 1)** On a clean machine with no `.env`, copy each portal's
   `.env.example` to `.env`, fill it in per the PR instructions, and run
   `yarn dev:creative-portal` and `yarn dev:customer-portal`. Both should start.
2. **(Issue 1)** Run `yarn test:e2e` for each portal and confirm the Playwright
   web server starts.
3. **(Issue 5)** On a devtest VM, confirm whether `MFA_DISABLED=true` is present.
   If it is, confirm the server still boots after this change — it will not.
4. **(Open Questions)** On each devtest / stage / production VM, print
   `NEXT_PUBLIC_ENVIRONMENT` and confirm it is exactly `devtest`, `stage` or
   `production` — lowercase, no surrounding whitespace.
5. **(Issue 12)** Confirm each live API key is at least 16 characters and that
   `NEXT_PUBLIC_GOOGLE_AUTH_CALLBACK` is an absolute `http(s)` URL.
6. **(Issue 10)** After a production build, confirm `.next/standalone` contains
   `yup`, `apps/*/env.js` and `packages/shared/utils/env-utils.js`.
7. **(Regression)** With `SERVICES_URI_REWRITE` unset, confirm both portals reach
   the backend exactly as before — service URIs should be byte-identical.
8. **(Feature)** With `SERVICES_URI_REWRITE` set locally, confirm both portals
   reach the Test backend and that API routes return live data.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Core logic sound; 5 confirmed defects in guard/validation edge cases |
| Regression risk | ❌ High — previously-soft failures are now process exits, and the deployed VM values are unverified |
| Tests | ⚠️ Large genuine improvement, but 4 key properties are untested and 18 tests are vacuous |
| Accessibility | n/a — no UI |
| Error handling | ⚠️ Fail-closed works, but one silent fail-open path and no Sentry on boot failure |
| Security | ⚠️ Net improvement over `develop`; guards are bypassable and the headline property is untested. `/security` was run by the author; re-run after the guard changes |
| Code quality | ✅ Conventions clean — Prettier, naming, reuse-first, `.js` precedent, no duplication |
| Validation suite | ⏭️ Skipped — user opted out; **and no CI checks ran on this PR** |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Request changes.**

This is careful, well-reasoned work that replaces validation which genuinely never
ran. The direction is right and most of it should land. But it converts a broad set
of soft failures into process exits while the repo contains no evidence of what the
deployment VMs actually hold, and the two safety properties the change is *named*
for — the guards and the host match — are respectively bypassable and untested.

**Before merge (blocking):**

1. Answer the first two Open Questions — the VM `NEXT_PUBLIC_ENVIRONMENT` values
   and whether the external build workflow exports `NEXT_PUBLIC_APP_VERSION`.
   Either one going the wrong way is a fleet-wide outage or a hard-blocked deploy
   with no bypass.
2. Fix Issue 1 — give `NEXT_PUBLIC_APP_VERSION` a working default in both
   `.env.example` files. Every reviewer hits this.
3. Fix Issue 2 — validate `NEXT_PUBLIC_ENVIRONMENT` before the skip branch and
   treat an unrecognised value as deployed.
4. Fix Issue 3 — add the suffix-direction test for the host match.
5. Fix Issue 4 — assert the throw in the customer-portal tests.
6. Resolve Issue 5 — make the code and the documentation agree on whether devtest
   is guarded, and confirm the devtest VMs before merging.

**Strongly recommended before merge:**

7. Issue 6 (trim whitespace), Issue 12 (drop the unsourced `.min(16)`), and
   Issue 7 (reuse the parser in the validator).
8. Re-run `/security` after the guard changes, and run the full validation suite —
   nothing has validated this branch in CI.

**Follow-ups (fine as separate tickets):** Issues 8, 10, 11, 13, 14, 15, and the
low-severity items 16–22.

Two things worth saying plainly: the comments in this PR are better than most
code in this repo, and the decision to build on yup rather than a bespoke
validator is the right one. The problems above are about the gap between what the
comments claim and what the code enforces — not about the approach.
