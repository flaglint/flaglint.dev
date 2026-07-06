---
title: "LaunchDarkly Feature Flag Cleanup: Audit, Rewrite, and Enforce in TypeScript"
description: "LaunchDarkly feature flag cleanup without the Enterprise tier: audit flag debt, rewrite call sites to OpenFeature, and enforce the boundary in CI — free OSS."
date: 2026-07-06
---

Your codebase has been accumulating direct LaunchDarkly SDK calls for years. The team knows a
cleanup is overdue, but nobody has a clear picture of how many flag keys exist, which ones are
safely automatable, and which ones will bite you if touched carelessly. LaunchDarkly's built-in
cleanup tooling — Vega — requires an Enterprise subscription. Grep finds string literals but
misses dynamic keys, detail evaluations, and bulk calls. You end up either doing nothing or
hand-editing files one at a time and hoping the argument order is correct.

[FlagLint](https://flaglint.dev) is a free, open-source CLI that automates
**LaunchDarkly feature flag cleanup** in TypeScript and JavaScript codebases using AST-based
static analysis — not regex — to classify every direct LaunchDarkly SDK call by risk, generate
a readable flag debt inventory, rewrite safe call sites to the OpenFeature standard, and enforce
the resulting boundary in CI. No LaunchDarkly API key required.

This guide walks through a complete cleanup cycle: audit → rewrite → enforce.

## Step 1: Audit your flag debt

The first step of any LaunchDarkly feature flag cleanup is understanding what you are dealing
with. Run `flaglint audit` against your source directory:

```bash
npx flaglint audit ./src
```

Here is the real output for a two-file Node.js service with seven flag evaluations across
checkout and pricing modules:

```
- Auditing ./src...
# FlagLint Audit Report

**Scanned at:** 2026-07-06T04:47:44.977Z
**Scan root:** ./src
**Files scanned:** 2
**Duration:** 35ms

## Summary

| Total Flags | High Risk | Medium Risk | Total Usages |
|-------------|-----------|-------------|--------------|
| 7 | 2 | 5 | 7 |

| Dynamic Keys | Detail Evals | Bulk Calls | Stale Signals | Safely Automatable | Manual Review |
|--------------|--------------|------------|---------------|-------------------|---------------|
| 1 | 1 | 0 | 0 | 5 | 2 |

> **Staleness:** No staleness signals detected. Heuristics checked: keyword match
> (flag key contains old/deprecated/legacy/temp/tmp/test/demo), path pattern
> (test/spec/mock files, deprecated/old/legacy directories), and minFileCount
> threshold. Git-history-based staleness (last evaluation date) requires git
> metadata and is not available in a pure static scan.

## Migration Readiness

Migration readiness: **71/100** · moderate

[██████████████████░░░░░░░] 71%

5 safely automatable  ·  2 require manual review

## Flag Debt Inventory

| Flag Key | Risk | Usages | Files | Call Types | Reasons |
|----------|------|--------|-------|------------|---------|
| `<dynamic key>` | 🔴 High | 1 | 1 | numberVariation | dynamic key |
| `beta-pricing` | 🔴 High | 1 | 1 | boolVariationDetail | detail evaluation |
| `checkout-v2` | 🟢 Automatable | 1 | 1 | boolVariation | safely automatable |
| `discount-percentage` | 🟢 Automatable | 1 | 1 | numberVariation | safely automatable |
| `ui-theme` | 🟢 Automatable | 1 | 1 | stringVariation | safely automatable |
| `checkout-config` | 🟡 Medium | 1 | 1 | jsonVariation | safely automatable, json variation |
| `promo-banner` | 🟢 Automatable | 1 | 1 | boolVariation | safely automatable |

## Next Steps

- Run `flaglint migrate --dry-run` to preview safe OpenFeature rewrites
- Run `flaglint validate --no-direct-launchdarkly` to enforce OF boundary in CI
- Review HIGH risk flags manually before any automated migration

✓ Audit complete: 7 flags — 2 high risk, 5 medium risk (35ms, 2 files)

Migration readiness: 71/100  ·  moderate
[██████████████████░░░░░░░] 71%
5 safely automatable  ·  2 require manual review
```

The readiness score of 71/100 means 5 of 7 call sites can be rewritten automatically. The two
high-risk entries need manual attention before anything else moves.

Add `--format html --output flag-debt.html` to produce a shareable report to attach to a
migration planning ticket. The [flag debt blog post](/blog/launchdarkly-flag-debt/) covers the
full range of audit options including effort estimates.

## What the flag debt inventory is telling you

Two call types are classified as high risk:

**Dynamic key** (`<dynamic key>`) — the flag key is constructed at runtime from a variable or
template literal (e.g., `` const key = `pricing-${plan}` ``). FlagLint cannot resolve which
flag is actually evaluated at any given call site. Any automated rewrite here would silently
touch the wrong call. These require a human decision: extract the dynamic key into a lookup
table, split into separate static flag keys, or handle manually per call type.

**Detail evaluation** (`beta-pricing` via `boolVariationDetail`) — `variationDetail` returns
a reason object with no direct OpenFeature equivalent. FlagLint skips these by design. You
need to decide whether that reason metadata is still needed after migration, and if so, which
OpenFeature detail API maps to your use case.

The five remaining call sites — `boolVariation`, `numberVariation`, `stringVariation`,
`jsonVariation`, and a second `boolVariation` — are all safely automatable. FlagLint can
rewrite every one of them without you touching a line.

## Step 2: Preview the safe rewrites

Run `flaglint migrate` with `--dry-run` to see exactly what changes before any file is
modified:

```bash
npx flaglint migrate ./src --dry-run
```

Real output (provider setup guidance section omitted; covered in Step 3 below):

```
- Scanning ./src...
LaunchDarkly usages found: 7
Safely automatable: 5 · Manual review: 2

Reviewable diffs: 5
Diffs requiring provider setup: 5
Skipped usages: 2

## Diffs
diff --git a/checkout.ts b/checkout.ts
--- a/checkout.ts
+++ b/checkout.ts
@@ -8,1 +8,1 @@
-  const newCheckoutEnabled = await ldClient.boolVariation("checkout-v2", ctx, false);
+  const newCheckoutEnabled = await openFeatureClient.getBooleanValue("checkout-v2", false, ctx);
@@ -9,1 +9,1 @@
-  const discountPct = await ldClient.numberVariation("discount-percentage", ctx, 0);
+  const discountPct = await openFeatureClient.getNumberValue("discount-percentage", 0, ctx);
@@ -10,1 +10,1 @@
-  const theme = await ldClient.stringVariation("ui-theme", ctx, "default");
+  const theme = await openFeatureClient.getStringValue("ui-theme", "default", ctx);
@@ -11,1 +11,1 @@
-  const config = await ldClient.jsonVariation("checkout-config", ctx, {});
+  const config = await openFeatureClient.getObjectValue("checkout-config", {}, ctx);

diff --git a/pricing.ts b/pricing.ts
--- a/pricing.ts
+++ b/pricing.ts
@@ -10,1 +10,1 @@
-  const promoEnabled = await ldClient.boolVariation("promo-banner", ctx, false);
+  const promoEnabled = await openFeatureClient.getBooleanValue("promo-banner", false, ctx);

## Skipped Usages
- pricing.ts:9:26 — `dynamicKey` via `numberVariation`: dynamic key requires manual review
- pricing.ts:11:23 — `beta-pricing` via `boolVariationDetail`: detail methods skipped:
  OpenFeature detail APIs exist, but LaunchDarkly/OpenFeature detail result parity requires
  manual review
```

Notice the argument order flip: `boolVariation("checkout-v2", ctx, false)` becomes
`getBooleanValue("checkout-v2", false, ctx)`. The LaunchDarkly SDK puts context second and
default last; OpenFeature reverses that. This reversed argument order is the [most common
source of silent production bugs in manual migrations](/blog/launchdarkly-openfeature-argument-order-bug/) —
FlagLint handles it correctly for every safe call type.

The `jsonVariation` → `getObjectValue` rewrite is flagged `json variation` in the audit
because OpenFeature's JSON type is `object`. If your LaunchDarkly flag ever returns a
primitive JSON value (number, string, boolean, null), call semantics differ. Review before
applying.

The two skipped usages are left exactly as-is in source.

## Step 3: Wire the OpenFeature provider

The dry-run output marks all five diffs as requiring provider setup. The LaunchDarkly SDK
stays as your evaluation backend — you are changing the API your application code calls, not
where flags are stored or evaluated.

Install once:

```bash
npm install @openfeature/server-sdk \
            @launchdarkly/node-server-sdk \
            @launchdarkly/openfeature-node-server
```

Add a bootstrap file at application startup. Do not remove existing LaunchDarkly packages —
the OpenFeature provider depends on them at runtime:

```typescript
import { OpenFeature } from "@openfeature/server-sdk";
import { LaunchDarklyProvider } from "@launchdarkly/openfeature-node-server";

const ldProvider = new LaunchDarklyProvider(process.env.LD_SDK_KEY!);
await OpenFeature.setProviderAndWait(ldProvider);

export const openFeatureClient = OpenFeature.getClient();
```

Import `openFeatureClient` in every module that has call sites in the migration plan, or
configure `openFeatureClientBindings` in your `.flaglintrc` so FlagLint locates the client
binding automatically. The [add OpenFeature provider tutorial](/docs/tutorials/add-openfeature-provider/)
covers both approaches with full examples.

## Step 4: Apply the safe rewrites

Once the OpenFeature provider is wired:

```bash
npx flaglint migrate ./src --apply
```

FlagLint rewrites only the five safely automatable call sites and leaves the two high-risk
ones untouched. What you get is an ordinary git diff: five function-call replacements across
two files, reviewable like any other PR. The dynamic key and detail evaluation remain as
direct LaunchDarkly SDK calls until you handle them manually.

## Step 5: Enforce the boundary in CI

After `--apply` and manual resolution of the remaining two call sites, lock the boundary so
no new direct LaunchDarkly SDK calls can reach main:

```bash
npx flaglint validate ./src --no-direct-launchdarkly
```

If you are mid-cleanup and cannot enforce a hard block yet, use baseline mode: it freezes
existing flag debt and fails on any net-new addition.

```bash
# Write current findings as the accepted baseline
npx flaglint audit ./src --write-baseline .flaglint-baseline.json

# In CI: fail only on findings not present in the baseline
npx flaglint validate ./src \
  --no-direct-launchdarkly \
  --baseline .flaglint-baseline.json \
  --fail-on-new
```

Commit `.flaglint-baseline.json` to source control. Each time you resolve a flag through
`migrate --apply` or a manual cleanup, re-run `--write-baseline` to shrink the accepted
set. The [GitHub Actions integration guide](/docs/integrations/github-actions/) shows the
full CI step configuration, including SARIF upload for GitHub Code Scanning annotations.

## Doing this at monorepo scale

If your LaunchDarkly SDK calls are spread across multiple packages, run the three commands
per package rather than at the repo root. Each package can have its own `.flaglintrc`
pointing to its local OpenFeature client binding. The
[monorepo guide](/docs/guides/monorepos/) covers per-package configuration and how to
sequence the cleanup when the same flag key is evaluated in shared libraries and consumer
apps simultaneously.

## Summary

LaunchDarkly feature flag cleanup at the code level breaks into four repeatable steps:

1. `flaglint audit ./src` — inventory your flag debt and get a readiness score
2. `flaglint migrate ./src --dry-run` — review the migration plan before touching files
3. `flaglint migrate ./src --apply` — apply safe rewrites; fix the remaining two manually
4. `flaglint validate ./src --no-direct-launchdarkly` — gate the boundary in CI

No Enterprise subscription, no API key, no manual grep. The
[complete six-step migration walkthrough](/docs/guides/launchdarkly-to-openfeature-nodejs/)
picks up from here if you want to see the full picture across a production Node.js service.
