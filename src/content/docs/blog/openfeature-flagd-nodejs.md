---
title: "OpenFeature flagd Node.js: Replace LaunchDarkly with a Self-Hosted Provider"
description: "Audit your LaunchDarkly flag debt with FlagLint and migrate to the OpenFeature flagd Node.js stack. Self-hosted, no vendor lock-in, zero SDK rewrites."
date: 2026-07-23
authors:
  - name: Krishan Sharma
    title: Founder and maintainer of FlagLint
    url: https://www.linkedin.com/in/krishansha/
tags: ["launchdarkly", "openfeature", "flagd", "nodejs", "migration"]
---

LaunchDarkly's pricing model has a well-documented inflection point. Feature flag evaluation is cheap per-call; access to the dashboard, the CLI, the audit log, and the enterprise tier isn't. Teams that hit the threshold face a classic build-or-buy rethink — often for the first time after years of letting the LaunchDarkly SDK grow roots across their Node.js codebase.

The migration is a two-phase problem, and each phase is independently reversible.

- **Phase 1 (code migration):** Use FlagLint to rewrite every direct LaunchDarkly SDK call to the OpenFeature API. Your flags keep evaluating through LaunchDarkly's backend — nothing breaks.
- **Phase 2 (provider swap):** Replace the LaunchDarkly OpenFeature provider with flagd and cut the vendor dependency entirely. One file changes.

This post walks both phases end-to-end using the OpenFeature flagd Node.js stack. If your application code already calls the OpenFeature API, jump straight to Phase 2.

## What the OpenFeature flagd Node.js stack gives you

[OpenFeature](https://openfeature.dev) is a CNCF-hosted specification for vendor-neutral flag evaluation. [flagd](https://flagd.dev) is its reference provider: a lightweight evaluation server you run yourself. It reads flag definitions from a JSON file, a Kubernetes ConfigMap, or a remote endpoint; evaluates them with fractional rollout and targeting rule support; and exposes a gRPC endpoint your application connects to.

The OpenFeature flagd Node.js combination means:

- **No vendor account.** flagd is a binary or Docker container you operate.
- **No pricing tier.** Flag count, MAU, team seats — none of it applies.
- **Portable flag definitions.** JSON files in your repo, versioned with your code.
- **Provider portability.** If you later want Flipt or GrowthBook, swapping providers is a one-file change — the same change you'll make when leaving LaunchDarkly.

## Phase 1: audit your flag debt

Point FlagLint at your source directory:

```bash
npx flaglint@latest audit ./src
```

If you accidentally point it at a directory with no JavaScript or TypeScript files — a documentation folder, for example — FlagLint tells you immediately:

```
- Auditing ./src/content/docs/...
No matching files found. Check your .flaglintrc include patterns.
```

Point it at your application source. Running against a Node.js service with seven LaunchDarkly flag call sites across two files produces:

```
- Auditing ....
# FlagLint Audit Report

**Scanned at:** 2026-07-23T03:06:31.585Z  
**Scan root:** /tmp/.../demo-app  
**Files scanned:** 2  
**Duration:** 45ms

## Summary

| Total Flags | High Risk | Medium Risk | Total Usages |
|-------------|-----------|-------------|--------------|
| 7 | 1 | 6 | 7 |

| Dynamic Keys | Detail Evals | Bulk Calls | Stale Signals | Safely Automatable | Manual Review |
|--------------|--------------|------------|---------------|-------------------|---------------|
| 0 | 0 | 1 | 0 | 6 | 1 |

> **Staleness:** No staleness signals detected. Heuristics checked: keyword match (flag key
> contains old/deprecated/legacy/temp/tmp/test/demo), path pattern (test/spec/mock files,
> deprecated/old/legacy directories), and minFileCount threshold.

## Migration Readiness

Migration readiness: **86/100** · ready

[██████████████████████░░░] 86%

6 safely automatable  ·  1 require manual review

## Flag Debt Inventory

| Flag Key | Risk | Usages | Files | Call Types | Reasons |
|----------|------|--------|-------|------------|---------|
| `*` | 🔴 High | 1 | 1 | allFlagsState | bulk call |
| `new-dashboard` | 🟢 Automatable | 1 | 1 | variation | safely automatable |
| `checkout-variant` | 🟢 Automatable | 1 | 1 | variation | safely automatable |
| `payment-v2` | 🟢 Automatable | 1 | 1 | variation | safely automatable |
| `rollout-percentage` | 🟢 Automatable | 1 | 1 | variation | safely automatable |
| `checkout-v2` | 🟢 Automatable | 1 | 1 | variation | safely automatable |
| `promo-banner` | 🟢 Automatable | 1 | 1 | variation | safely automatable |

## Next Steps

- Run `flaglint migrate --dry-run` to preview safe OpenFeature rewrites
- Run `flaglint validate --no-direct-launchdarkly` to enforce OF boundary in CI
- Review HIGH risk flags manually before any automated migration

✓ Audit complete: 7 flags — 1 high risk, 6 medium risk (45ms, 2 files)
```

A readiness score of **86/100** means six call sites are safely automatable. The one high-risk item — `allFlagsState` — is a bulk inventory call with no single-flag codemod. [Five LaunchDarkly SDK Patterns That Block Automatic Migration](/blog/five-patterns-that-block-migration) covers how to resolve it manually.

For a larger service, add `--effort-estimate` to get a migration hours projection before you commit to the work. See [LaunchDarkly Flag Debt](/blog/launchdarkly-flag-debt) for how to read the output.

## Phase 1: preview the rewrites

Before writing anything to disk, preview what changes:

```bash
npx flaglint@latest migrate --dry-run ./src
```

```
LaunchDarkly usages found: 7
Safely automatable: 6 · Manual review: 1
# FlagLint migrate --dry-run

These diffs use the placeholder `openFeatureClient` and require OpenFeature
provider/client setup before they can be applied.
No files are modified by dry-run output.

Reviewable diffs: 6
Diffs requiring provider setup: 6
Skipped usages: 1

## Diffs
diff --git a/featureFlags.ts b/featureFlags.ts
--- a/featureFlags.ts
+++ b/featureFlags.ts
@@ -7,1 +7,1 @@
-  return client.variation('new-dashboard', user, false);
+  return openFeatureClient.getBooleanValue('new-dashboard', false, user);
@@ -12,1 +12,1 @@
-  return client.variation('checkout-variant', user, 'control');
+  return openFeatureClient.getStringValue('checkout-variant', 'control', user);
@@ -17,1 +17,1 @@
-  return client.variation('payment-v2', context, false);
+  return openFeatureClient.getBooleanValue('payment-v2', false, context);
@@ -22,1 +22,1 @@
-  return client.variation('rollout-percentage', user, 0);
+  return openFeatureClient.getNumberValue('rollout-percentage', 0, user);

diff --git a/routes.ts b/routes.ts
--- a/routes.ts
+++ b/routes.ts
@@ -7,1 +7,1 @@
-  const useNewFlow = await ldClient.variation('checkout-v2', user, false);
+  const useNewFlow = await openFeatureClient.getBooleanValue('checkout-v2', false, user);
@@ -8,1 +8,1 @@
-  const showBanner = await ldClient.variation('promo-banner', user, false);
+  const showBanner = await openFeatureClient.getBooleanValue('promo-banner', false, user);

## Skipped Usages
- routes.ts:18:9 — `*` via `allFlagsState`: bulk inventory call has no single-flag codemod
```

Notice the argument order shift: `variation(flag key, context, default)` becomes `getBooleanValue(flag key, default, context)`. FlagLint rewrites this at the AST level — not with find-and-replace — so it handles wrappers and aliased clients that grep misses. Getting the argument order wrong silently is the most common production bug in LaunchDarkly-to-OpenFeature migrations. See [Why LaunchDarkly → OpenFeature Migrations Break in Production](/blog/launchdarkly-openfeature-argument-order-bug) for the full explanation.

## Phase 1: apply with the LaunchDarkly OpenFeature provider

Apply the rewrites now, while keeping LaunchDarkly evaluating your flags. Flags continue to work exactly as before; you get to run your test suite before touching the backend at all.

Install the provider packages:

```bash
npm install @openfeature/server-sdk @launchdarkly/node-server-sdk @launchdarkly/openfeature-node-server
```

Add provider initialization at application startup:

```typescript
// bootstrap.ts
import { OpenFeature } from "@openfeature/server-sdk";
import { LaunchDarklyProvider } from "@launchdarkly/openfeature-node-server";

const provider = new LaunchDarklyProvider(process.env.LD_SDK_KEY!);
await OpenFeature.setProviderAndWait(provider);

export const openFeatureClient = OpenFeature.getClient();
```

Apply the rewrites:

```bash
npx flaglint@latest migrate ./src
```

Your call sites now use `getBooleanValue`, `getStringValue`, and `getNumberValue` from the OpenFeature server SDK. The LaunchDarkly SDK key is still required at runtime — the LaunchDarkly OpenFeature provider wraps it. But your application code no longer imports from `@launchdarkly/node-server-sdk` directly. You can replace the provider later without touching a single call site.

## Phase 2: swap to flagd

Once the OpenFeature migration is complete and your test suite passes, swapping the backend from LaunchDarkly to flagd takes two steps: write your flag definitions, then update the single bootstrap file.

### Start flagd

```bash
# macOS / Linux via Homebrew
brew install open-feature/tap/flagd
flagd start --uri file:./flags.json

# Docker
docker run -p 8013:8013 ghcr.io/open-feature/flagd:latest start --uri file:./flags.json
```

### Write flag definitions

```json
{
  "$schema": "https://flagd.dev/schema/v0/flags.json",
  "flags": {
    "new-dashboard": {
      "state": "ENABLED",
      "variants": { "on": true, "off": false },
      "defaultVariant": "on"
    },
    "checkout-variant": {
      "state": "ENABLED",
      "variants": { "control": "control", "v2": "v2" },
      "defaultVariant": "control"
    },
    "checkout-v2": {
      "state": "ENABLED",
      "variants": { "on": true, "off": false },
      "defaultVariant": "off"
    },
    "promo-banner": {
      "state": "DISABLED",
      "variants": { "on": true, "off": false },
      "defaultVariant": "off"
    }
  }
}
```

Commit this file to your repository. Flag changes are now a JSON diff in a pull request — reviewable, auditable, and rolled back with a revert.

### Update bootstrap.ts — the only file that changes

```bash
npm install @openfeature/server-sdk @openfeature/flagd-provider
```

```typescript
// bootstrap.ts — only this file changes
import { OpenFeature } from "@openfeature/server-sdk";
import { FlagdProvider } from "@openfeature/flagd-provider";

await OpenFeature.setProviderAndWait(new FlagdProvider());

export const openFeatureClient = OpenFeature.getClient();
```

Delete `LD_SDK_KEY` from your environment. Every flag evaluation in your application now routes to flagd. Every call site you migrated in Phase 1 is unchanged.

This is the payoff of the OpenFeature flagd Node.js approach: provider-agnostic application code where the backend is genuinely swappable. If you later want percentage rollouts or targeting rules, flagd's JSON schema supports both without changing application code. If you want to move to a hosted provider, the swap is again a one-file change.

## Enforce the boundary in CI

Lock in the migration by blocking new direct LaunchDarkly SDK calls from landing:

```bash
npx flaglint@latest validate --no-direct-launchdarkly ./src
```

Add this step to your GitHub Actions workflow:

```yaml
- name: Enforce OpenFeature boundary
  run: npx flaglint@latest validate --no-direct-launchdarkly ./src
```

Full setup with SARIF annotation upload is covered in [Enforcing Your LaunchDarkly to OpenFeature Migration in GitHub Actions](/blog/enforce-launchdarkly-migration-github-actions).

## Next steps

- Complete end-to-end workflow including CI baseline mode: [LaunchDarkly to OpenFeature Node.js Migration Guide](/guides/launchdarkly-to-openfeature-nodejs).
- For the high-risk `allFlagsState` call in the audit above, see [Five Patterns That Block Automatic Migration](/blog/five-patterns-that-block-migration).
- Estimate migration hours before starting: `flaglint audit --effort-estimate --hourly-rate 125 ./src`. [LaunchDarkly Flag Debt](/blog/launchdarkly-flag-debt) explains how to read the output.

The OpenFeature flagd Node.js stack resolves both categories of LaunchDarkly dependency: the SDK lock-in in your code (FlagLint handles Phase 1), and the vendor dependency in your infrastructure (flagd handles Phase 2). Phase 1 takes an afternoon for a typical Node.js service. Phase 2 is one file change and a bootstrap swap.
