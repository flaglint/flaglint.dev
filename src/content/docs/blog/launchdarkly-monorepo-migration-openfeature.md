---
title: "LaunchDarkly Monorepo Migration to OpenFeature: Migrate One Package at a Time"
description: "LaunchDarkly monorepo migration to OpenFeature: audit each service with FlagLint, preview safe rewrites, and enforce the boundary in CI. Free OSS, no API key."
date: 2026-07-09
---

Your monorepo has a dozen services, a shared feature-flag wrapper, and three years of direct
LaunchDarkly SDK calls spread across packages. You've decided to move to OpenFeature, but nobody
wants to touch fifteen services at once. One broken migration branch stalls the entire team.

The right approach for a LaunchDarkly monorepo migration is incremental: audit one service,
preview the rewrites, apply, enforce in CI, move to the next. [FlagLint](https://flaglint.dev)
is designed for exactly this pattern — it operates on a single directory per invocation and reads
a per-package `.flaglintrc`, so audit and migrate commands stay scoped to one service's conventions
without touching anything else.

This guide walks through the full cycle for one service in a Turborepo or Nx workspace.

## Why monorepos complicate flag migration

A single-package repo has one import path for the LaunchDarkly SDK, one shared client, and one
set of call sites. A monorepo doesn't.

**Import paths diverge per package.** `services/checkout` might import `@company/feature-flags`.
`services/pricing` imports a different internal wrapper. Both ultimately call the LaunchDarkly SDK,
but through different module paths.

**Shared clients live in internal packages.** A `packages/feature-flag-client` workspace package
often exports an OpenFeature-shaped wrapper around the LaunchDarkly SDK. FlagLint needs to know
this binding to classify call sites correctly — otherwise it reports false flag debt on wrapper
calls that are already vendor-neutral.

**Migration risk varies by service.** One service evaluates only simple boolean flag keys — every
call site is automatable. Another has dynamic key construction and bulk `allFlagsState` calls.
Your migration plan should reflect that spread before you write a single line.

Running any migration tool at the workspace root treats all of that as uniform. A
LaunchDarkly monorepo migration needs service-level granularity.

## Workspace structure

A typical Turborepo layout:

```text
services/
  checkout/
    src/
      routes/checkout.ts
      platform/feature-flags.ts
    .flaglintrc
    package.json
  pricing/
    src/
    .flaglintrc
packages/
  feature-flag-client/
    src/
      index.ts           # shared OpenFeature wrapper
```

Each service carries its own `.flaglintrc`. FlagLint reads that file when you pass `--config`,
keeping every command scoped to one package's import conventions.

## Step 1 — Audit one service

Start with the service you know best. Run `flaglint audit` against its source directory:

```bash
npx flaglint audit ./services/checkout/src \
  --config ./services/checkout/.flaglintrc
```

The output is a flag debt inventory: every call site classified by call type (boolean variation,
string variation, JSON variation, detail evaluation), with a readiness score for each that tells
you whether FlagLint can rewrite it automatically or whether it needs manual review.

Here is what FlagLint reports when a source directory contains no LaunchDarkly SDK usage:

```
- Auditing ./src/...
No LaunchDarkly SDK usage detected in 1 files.
```

That is your target state for every service. Once a package reaches that output, it carries zero
flag debt and the LaunchDarkly SDK dependency can be removed from that package entirely.

## Step 2 — Per-package `.flaglintrc`

Create a `.flaglintrc` in each service root. The most important section for monorepos is
`openFeatureClientBindings`, which tells FlagLint how the OpenFeature client is imported in
that package:

```json
// services/checkout/.flaglintrc
{
  "include": ["**/*.{ts,js}"],
  "exclude": [
    "**/node_modules/**",
    "**/dist/**",
    "**/*.test.ts",
    "**/*.spec.ts"
  ],
  "openFeatureClientBindings": [
    {
      "importName": "openFeatureClient",
      "modulePatterns": ["**/platform/feature-flags"]
    }
  ]
}
```

If your shared client lives in a workspace package, add its import pattern:

```json
{
  "openFeatureClientBindings": [
    {
      "importName": "featureFlagClient",
      "modulePatterns": ["**/feature-flag-client/src/index"]
    }
  ]
}
```

Without this, FlagLint cannot distinguish a direct LaunchDarkly SDK call from a call through your
internal wrapper. The audit reports false flag debt on wrapper call sites, inflating your
readiness score estimate and causing the rewriter to touch files it shouldn't.

For the full per-package configuration reference, see the [monorepo guide](/docs/guides/monorepos/).

## Step 3 — Preview the rewrites

Before writing any file, run `flaglint migrate` in dry-run mode. This produces the exact
before/after diff for every call site FlagLint can safely rewrite, without touching disk:

```bash
npx flaglint migrate ./services/checkout/src \
  --config ./services/checkout/.flaglintrc \
  --dry-run
```

The diff shows three things per call site: the flag key being migrated, the call type being
replaced, and the [OpenFeature provider](/docs/integrations/openfeature-provider/) equivalent
it becomes.

Any call site that carries a staleness signal — a flag key constructed at runtime, a
`variationDetail` call, or a bulk `allFlagsState` call — is marked for manual review rather
than automated rewrite. The stale signal classification is carried through from the audit; the
dry-run makes it visible in diff form before you commit to anything.

Check the diff carefully. If it touches more files than the audit suggested, review your
`openFeatureClientBindings` pattern first — a too-broad module pattern causes over-reporting.
The readiness score in the audit tells you what percentage of call sites are automatable; the
dry-run confirms the exact set.

## Step 4 — Apply per package on a branch

Open a branch scoped to this one service, then apply:

```bash
git checkout -b migrate/checkout-openfeature

npx flaglint migrate ./services/checkout/src \
  --config ./services/checkout/.flaglintrc \
  --apply
```

FlagLint rewrites only the call sites it classified as automatable. Each rewrite replaces a
direct LaunchDarkly SDK call with the OpenFeature equivalent, preserving flag key, fallback
value, and return type. The argument-order difference — LaunchDarkly puts the default value
last; OpenFeature puts it second — is handled by the rewriter. This is the class of bug that
grep-based migration scripts consistently miss. See
[why argument order breaks production migrations](/blog/launchdarkly-openfeature-argument-order-bug/)
for the detailed breakdown.

Verify the service starts and its tests pass before opening the PR. Keep the migration branch
scoped to one service — nothing else changes.

## Step 5 — Enforce in CI per service

Once a service's migration branch merges, lock it. The `flaglint validate` command fails the
build if any new direct LaunchDarkly SDK call appears in that source directory:

```bash
npx flaglint validate ./services/checkout/src \
  --config ./services/checkout/.flaglintrc \
  --no-direct-launchdarkly
```

In GitHub Actions, run this as a matrix across migrated services:

```yaml
name: OpenFeature boundary check
on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        package: [checkout, pricing, inventory]
    steps:
      - uses: actions/checkout@v4
      - name: Enforce OpenFeature boundary — ${{ matrix.package }}
        run: |
          npx flaglint validate ./services/${{ matrix.package }}/src \
            --config ./services/${{ matrix.package }}/.flaglintrc \
            --no-direct-launchdarkly
```

Add a service to the matrix as soon as its migration branch merges. From that point forward, CI
blocks any regression to the LaunchDarkly SDK on that service. For a complete CI walkthrough, see
[enforcing your migration in GitHub Actions](/blog/enforce-launchdarkly-migration-github-actions/).

## What stays manual

Four call types require hand-editing and are excluded from automated rewrites:

**Dynamic flag keys** — `ldClient.variation(buildKey(userId, 'checkout-v2'), ...)` cannot be
safely rewritten. The flag key is not a string literal at the call site. FlagLint marks it with
a stale signal and surfaces it for manual review.

**`allFlagsState` calls** — No direct OpenFeature provider equivalent. These are typically used
for bootstrapping client-side applications and require an architectural decision before
rewriting.

**`variationDetail` calls** — Returns a reason object with no standard equivalent across
OpenFeature providers. Plan this call type explicitly in your migration plan before touching it.

**Configured wrappers with non-standard argument order** — If your internal wrapper swaps the
flag key and default value positions compared to the LaunchDarkly SDK, the rewriter detects a
staleness signal and surfaces the call site for review rather than rewriting it automatically.

The [manual review patterns guide](/docs/guides/manual-review-patterns/) covers each of these
with before/after examples.

## Sequencing the full migration

A LaunchDarkly monorepo migration is a sequence, not a parallel operation. A practical order:

1. Run `flaglint audit` across all services. Rank by readiness score — highest first.
2. Migrate the highest-readiness service. Enforce in CI before moving to the next.
3. For each subsequent service, run `--dry-run` to estimate effort before scheduling the work.
4. For services with manual call types, open a tracking issue listing the flag keys that need
   hand-editing.
5. Once every service passes `flaglint validate --no-direct-launchdarkly`, remove the
   LaunchDarkly SDK from the workspace root `package.json`.

Teams that have tried to migrate all services in one branch report the same three failure modes:
merge conflicts across services, a broken shared client that blocks everything, and a diff so
large that reviewers stop reviewing. The per-service approach eliminates all three. Each merged
branch is a verified, independently deployable step — the migration history is visible in your
commit graph.

For a per-package configuration reference including workspace-level CI patterns, see the
[monorepo guide](/docs/guides/monorepos/).
