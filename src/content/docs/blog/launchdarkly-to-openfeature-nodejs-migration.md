---
title: "How to Migrate from LaunchDarkly to OpenFeature in Node.js"
date: 2026-07-05
description: "A step-by-step guide to migrating your Node.js service from the LaunchDarkly SDK to OpenFeature — without breaking flag evaluations in production. Covers audit, provider setup, safe rewrites, and CI enforcement."
authors:
  - name: Krishan Sharma
    title: Founder and maintainer of FlagLint
    url: https://www.linkedin.com/in/krishansha/
tags: ["launchdarkly", "openfeature", "migration", "nodejs"]
---

Most LaunchDarkly to OpenFeature migrations start the same way: someone opens a PR
with a find-and-replace across the codebase, the tests pass, the PR merges, and
two days later a subset of users hits the wrong feature state in production.

The root cause is almost always the same one-line trap — and this guide shows you
how to avoid it entirely, using a workflow that audits first, rewrites only what
it can prove is safe, and enforces the boundary in CI when you're done.

<!-- excerpt -->

## The one trap that breaks every naive migration

LaunchDarkly and OpenFeature both evaluate flags with three arguments. The
argument order is different:

```typescript
// LaunchDarkly: key → context → fallback
ldClient.boolVariation("checkout-v2", context, false)

// OpenFeature: key → fallback → context
openFeatureClient.getBooleanValue("checkout-v2", false, context)
```

A find-and-replace or a naive codemod swaps `context` and `fallback` silently.
The code compiles. The tests pass (your test context is usually a dummy object
and your fallback is usually `false` or `"default"`). In production, flags
evaluate against a boolean where they expected a user context object, and the
evaluation falls back every time.

This is not an edge case — it is the most common migration bug we see. The
[argument-order post](/blog/launchdarkly-openfeature-argument-order-bug/) covers
it in depth. The short version: **don't use find-and-replace for this migration.**

---

## The right workflow: audit → migrate → enforce

FlagLint is a free, open-source CLI that gives you a safe, reviewable migration
in three stages:

1. **Audit** — understand what you have before touching anything
2. **Migrate** — generate and apply only proven-safe rewrites
3. **Enforce** — block new direct LaunchDarkly calls in CI

Nothing in your codebase changes until you explicitly run `--apply`. Every step
is local: no source upload, no API key.

---

## Step 1: Audit your codebase

```bash
npx flaglint audit ./src
```

This scans every file under `./src` for direct LaunchDarkly SDK calls and
produces a risk-ranked report:

```
✓ Audit complete: 13 unique flags across 19 call sites — 3 high risk, 10 medium risk

  Readiness: 53/100 · moderate
  10 of 19 call sites safely automatable · 9 require manual review

Flag Key               Risk     Usages  Reason
────────────────────────────────────────────────────
<dynamic key>          High        8   dynamic key
checkout-experiment    High        1   detail evaluation
*                      High        1   bulk call
checkout-v2            Medium      1   safely automatable
payment-provider       Medium      1   safely automatable
```

The readiness score and the automatable count tell you the shape of the work
before you write a single line. The high-risk flags are the ones that require
manual review — more on those below.

For a full HTML report with call-site locations:

```bash
npx flaglint audit ./src --format html --output report.html
```

---

## Step 2: Set up the OpenFeature provider (once)

FlagLint migrates the call-site API — the LaunchDarkly backend stays your
provider, it just talks to it through the OpenFeature interface. You need to
wire up the provider once, before running `--apply`.

Install the packages:

```bash
npm install @openfeature/server-sdk @launchdarkly/openfeature-node-server
```

Bootstrap at application startup (before any flag evaluation):

```typescript
import { OpenFeature } from "@openfeature/server-sdk";
import { LaunchDarklyProvider } from "@launchdarkly/openfeature-node-server";

await OpenFeature.setProviderAndWait(
  new LaunchDarklyProvider(process.env.LD_SDK_KEY!)
);
export const openFeatureClient = OpenFeature.getClient();
```

Keep the LaunchDarkly packages — the OpenFeature provider depends on them at
runtime. You're not removing LaunchDarkly; you're migrating the API your
application code calls.

---

## Step 3: Preview the migration with dry-run

```bash
npx flaglint migrate ./src --dry-run
```

This prints a reviewable diff for every call site FlagLint can safely rewrite —
without changing any files:

```diff
--- a/src/routes/checkout.ts
+++ b/src/routes/checkout.ts
- const enabled = await ldClient.boolVariation("checkout-v2", ctx, false);
+ const enabled = await openFeatureClient.getBooleanValue("checkout-v2", false, ctx);

--- a/src/services/pricing.ts
+++ b/src/services/pricing.ts
- const provider = await ldClient.stringVariation("payment-provider", ctx, "stripe");
+ const provider = await openFeatureClient.getStringValue("payment-provider", "stripe", ctx);
```

Notice the argument order is corrected automatically. FlagLint verified the flag
key is static, the fallback type is known, and your OpenFeature client binding
is in scope — only then does it generate the rewrite.

The dry-run also lists the calls it will **not** touch:

```
skip: src/services/experiments.ts:42 — dynamic key cannot be resolved statically
skip: src/routes/checkout.ts:88     — boolVariationDetail (detail evaluation)
```

Review the diff. If anything looks wrong, fix it before applying. The tool does
not change files until you say so.

---

## Step 4: Apply the migration

```bash
npx flaglint migrate ./src --apply
```

This writes the proven-safe rewrites to disk. Commit the result as a single
reviewable PR. The diff will look exactly like the dry-run output — no surprises.

Run your test suite. If a test fails, it will be on one of the skipped calls
(dynamic keys, detail evaluations, bulk calls) — the auto-rewritten calls are
proven correct by construction.

---

## Step 5: Handle the manual-review cases

The calls FlagLint skipped are the ones where a mechanical rewrite would be
unsafe. These require human judgment:

| Pattern | Why it's skipped | What to do |
|---------|-----------------|------------|
| Dynamic keys | `ldClient.boolVariation(getFlagKey(user), ...)` — key isn't a literal | Refactor to extract the key as a static constant |
| Detail evaluations | `boolVariationDetail`, `variationDetail` — returns value + reason/metadata | Use `openFeatureClient.getBooleanDetails()` to get both value and reason |
| Bulk calls | `allFlags()`, `allFlagsState()` — no OpenFeature equivalent | Evaluate each flag individually or keep the LD call isolated |
| Ambiguous binding | Multiple OpenFeature clients in scope | Refactor to a shared singleton so only one client variable is in scope |

In practice, dynamic keys and detail evaluations account for most skips.
The [five patterns post](/blog/five-patterns-that-block-migration/) covers each
one in detail.

---

## Step 6: Enforce the boundary in CI

Once the migration is done, add a validation step to your CI pipeline to block
any new direct LaunchDarkly calls from being merged:

```bash
npx flaglint validate ./src --no-direct-launchdarkly
```

This exits with code 1 if any direct `ldClient.*Variation` calls are found.
Add it to your GitHub Actions workflow:

```yaml
- name: Enforce OpenFeature boundary
  run: npx flaglint validate ./src --no-direct-launchdarkly
```

For pull request line annotations, use `--format sarif` and upload to GitHub
Code Scanning:

```yaml
- name: Validate OpenFeature boundary
  run: npx flaglint validate ./src --no-direct-launchdarkly --format sarif --output flaglint.sarif
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: flaglint.sarif
```

The validation step is the last thing you add and the first thing to notice if
someone reintroduces a direct LaunchDarkly call three months from now.

---

## What doesn't change

- Your LaunchDarkly account and flag configuration stay exactly as they are.
- The LaunchDarkly SDK stays in your `package.json` — the OpenFeature provider
  depends on it.
- Evaluation context structure is compatible: OpenFeature accepts `targetingKey`
  or an existing LaunchDarkly `key` field.
- The LaunchDarkly dashboard, targeting rules, and rollout percentages work
  identically through the OpenFeature provider.

The migration changes only the API your application code calls — not the
system that evaluates the flags.

---

## Full workflow summary

```bash
# 1. See what you have
npx flaglint audit ./src

# 2. Set up OpenFeature provider (one-time, manual — see above)

# 3. Preview rewrites — no files changed
npx flaglint migrate ./src --dry-run

# 4. Apply proven-safe rewrites
npx flaglint migrate ./src --apply

# 5. Handle the skipped cases manually

# 6. Lock it down in CI
npx flaglint validate ./src --no-direct-launchdarkly
```

No API key. No source upload. Everything runs locally in your checkout.

---

## Next steps

- [Quickstart](/docs/quickstart/) — install and run your first audit in two minutes
- [Full migration guide](/docs/tutorials/migrate-a-node-service/) — deeper walkthrough with a real service
- [Safety model](/docs/concepts/safety-model/) — exactly what FlagLint proves before rewriting
- [Five patterns that block migration](/blog/five-patterns-that-block-migration/) — handling the manual-review cases
