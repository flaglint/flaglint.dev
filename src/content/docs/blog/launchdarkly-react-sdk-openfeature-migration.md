---
title: "LaunchDarkly React SDK Migration to OpenFeature: A Manual-Review Playbook"
description: "FlagLint audits your LaunchDarkly React SDK migration: detect useFlags and useVariation hooks, generate before/after diffs, and enforce OpenFeature in CI."
date: 2026-07-16
authors:
  - name: Krishan Sharma
    title: Founder and maintainer of FlagLint
    url: https://www.linkedin.com/in/krishansha/
tags: ["launchdarkly", "openfeature", "react", "migration", "hooks"]
---

A LaunchDarkly React SDK migration is a different problem from migrating a Node.js service. When
you run `flaglint migrate --apply` on a server-side codebase, it rewrites `ldClient.boolVariation`
call sites automatically. React is different. The LaunchDarkly React SDK ships a hooks-based
evaluation API — `useFlags`, `useVariation`, `useLDClient` — that is structurally coupled to
component trees and context providers. FlagLint detects every call site and surfaces them in the
migration plan, but the rewrites are flagged as manual review items, not auto-applied.

This article is for engineers who have run FlagLint on a React codebase and are looking at a list
of manual review items. It covers what each hook maps to in the OpenFeature React SDK, shows
real before/after diffs for each conversion, and explains how to lock the boundary in CI once
migration is complete.

<!-- excerpt -->

## Start with the audit

Before touching any component, get the complete flag debt inventory. Run the audit command
against your source directory:

```bash
npx flaglint@latest audit ./src/content/docs/
```

On the FlagLint docs site itself — which has no TypeScript SDK calls — the real output is:

```
- Auditing ./src/content/docs/...
No matching files found. Check your .flaglintrc include patterns.
```

On a real React app, point the command at your source:

```bash
npx flaglint@latest audit ./src
```

The audit report lists every direct LaunchDarkly SDK call, classified by risk level and call type.
React SDK hooks appear in the manual review section. The readiness score reflects what fraction of
call sites are safely automatable — React hooks count against it because FlagLint cannot
auto-rewrite them without risking context provider breakage. Use the
[flag debt audit guide](/blog/launchdarkly-flag-debt) if you want to include an
effort estimate alongside the inventory.

To get the raw inventory without the report format, use `scan`:

```bash
npx flaglint@latest scan ./src
```

This lists every flag key, call type, and staleness signal across your codebase. The scan output
is the input you'll work from as you convert hooks one file at a time.

---

## Why React hooks are manual review items

FlagLint auto-applies rewrites on the server-side Node.js SDK only. The conservative boundary is
intentional: React SDK hooks are entangled with the component tree in ways that static analysis
cannot safely resolve.

- `useFlags()` returns a dictionary of all flag values. There is no OpenFeature equivalent — the
  conversion requires splitting one call into one `useFlag` call per flag key, which is a
  component-level judgment.
- `useLDClient()` exposes the raw LaunchDarkly client. A mechanical replacement with
  `useOpenFeatureClient()` would compile, but any downstream method calls on the client
  object (track events, custom attributes) may have no OpenFeature analog.
- Context provider wrappers (`LDProvider`, `withLDProvider`) are bootstrap infrastructure;
  removing them before all hook call sites are converted will cause runtime failures.

The [five patterns that block automatic migration](/blog/five-patterns-that-block-migration) covers
the server-side equivalents in detail. The React hook situation is a separate category: these
patterns are not blocked because they are ambiguous — they are blocked because component trees
require human judgment to restructure.

---

## Install the OpenFeature React SDK

The OpenFeature JavaScript ecosystem splits into server-side and web (client-side) SDKs. For
React, you need the web SDK:

```bash
npm install @openfeature/react-sdk
```

Choose a provider that wraps your existing backend. If you are keeping LaunchDarkly as the flag
evaluation backend, the official LaunchDarkly OpenFeature web provider is the right choice:

```bash
npm install @launchdarkly/openfeature-web-provider
```

For a full list of providers and server-side setup, see the
[OpenFeature provider setup guide](/docs/integrations/openfeature-provider).

Bootstrap once at the root of the component tree:

```tsx
// app/layout.tsx (Next.js App Router) or _app.tsx
import { OpenFeatureProvider, OpenFeature } from '@openfeature/react-sdk'
import { LaunchDarklyWebProvider } from '@launchdarkly/openfeature-web-provider'

const provider = new LaunchDarklyWebProvider(
  process.env.NEXT_PUBLIC_LD_CLIENT_SIDE_ID!,
  { key: currentUser.id }
)

await OpenFeature.setProviderAndWait(provider)

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <OpenFeatureProvider>{children}</OpenFeatureProvider>
}
```

This is the only place that retains a reference to the LaunchDarkly SDK. Every component below it
evaluates flags through the OpenFeature API.

---

## Before/after: useFlags

`useFlags()` returns a dictionary of every flag value in the current evaluation context. The
OpenFeature React SDK has no equivalent — you call `useFlag` once per flag key.

**Before (LaunchDarkly React SDK):**

```tsx
import { useFlags } from 'launchdarkly-react-client-sdk'

function PricingPage() {
  const { showAnnualPricing, enableDiscountBanner } = useFlags()

  return (
    <>
      {showAnnualPricing && <AnnualPricingPanel />}
      {enableDiscountBanner && <DiscountBanner />}
    </>
  )
}
```

**After (OpenFeature React SDK):**

```tsx
import { useFlag } from '@openfeature/react-sdk'

function PricingPage() {
  const { value: showAnnualPricing } = useFlag('show-annual-pricing', false)
  const { value: enableDiscountBanner } = useFlag('enable-discount-banner', false)

  return (
    <>
      {showAnnualPricing && <AnnualPricingPanel />}
      {enableDiscountBanner && <DiscountBanner />}
    </>
  )
}
```

One `useFlag` call per flag key. The `flaglint scan` output gives you the full list of flag keys
used in each file — use it to verify you haven't missed any from a `useFlags` destructure.

Note the flag key format: LaunchDarkly's React SDK typically uses camelCase keys via automatic
transformation. The OpenFeature SDK uses the raw key as stored in your flag management backend,
which is usually kebab-case. Check your flag keys in LaunchDarkly's dashboard before converting.

---

## Before/after: useVariation

`useVariation(flagKey, defaultValue)` maps directly to `useFlag(flagKey, defaultValue)`. The call
signature is almost identical; only the return shape differs.

**Before:**

```tsx
import { useVariation } from 'launchdarkly-react-client-sdk'

function CheckoutButton() {
  const newCheckoutEnabled = useVariation('new-checkout-flow', false)
  return newCheckoutEnabled ? <NewCheckout /> : <LegacyCheckout />
}
```

**After:**

```tsx
import { useFlag } from '@openfeature/react-sdk'

function CheckoutButton() {
  const { value: newCheckoutEnabled } = useFlag('new-checkout-flow', false)
  return newCheckoutEnabled ? <NewCheckout /> : <LegacyCheckout />
}
```

Destructure `value` from the result object. OpenFeature also exposes `reason`, `errorCode`, and
`flagMetadata` from `useFlag` — useful if you want to surface evaluation details in logging or
observability tooling.

---

## Before/after: useLDClient

`useLDClient()` returns the raw LaunchDarkly client instance, typically used for initialization
checks or manual `track` calls. OpenFeature's equivalent is `useOpenFeatureClient()`.

**Before:**

```tsx
import { useLDClient } from 'launchdarkly-react-client-sdk'

function FeatureGate({ children }: { children: React.ReactNode }) {
  const ldClient = useLDClient()
  if (!ldClient) return null
  return <>{children}</>
}
```

**After:**

```tsx
import { useOpenFeatureClient } from '@openfeature/react-sdk'

function FeatureGate({ children }: { children: React.ReactNode }) {
  const client = useOpenFeatureClient()
  if (!client) return null
  return <>{children}</>
}
```

If you were calling `ldClient.track()` for experimentation events, check your OpenFeature
provider's documentation — OpenFeature does not define a standard track method. LaunchDarkly's
OpenFeature web provider may expose track via a custom domain API.

---

## Remove LDProvider last

The LaunchDarkly React SDK wraps the app in one of two provider components — `LDProvider` or
`withLDProvider`. These are bootstrap infrastructure, not flag evaluation call sites.

Delete them **last**, after every `useFlags`, `useVariation`, and `useLDClient` call site has been
converted and tested. Removing the provider before converting hooks will produce runtime errors as
components attempt to access a non-existent LaunchDarkly context.

Once `LDProvider` / `withLDProvider` is removed, uninstall the React SDK package:

```bash
npm uninstall launchdarkly-react-client-sdk
```

If you were using `@launchdarkly/react-client-sdk` (the renamed package), uninstall that instead.

---

## Enforce the boundary in CI

Once the LaunchDarkly React SDK migration is complete, prevent regressions with `flaglint validate`:

```bash
npx flaglint@latest validate ./src --no-direct-launchdarkly
```

The `--no-direct-launchdarkly` flag makes the command exit non-zero if any direct LaunchDarkly
SDK import is found — including `launchdarkly-react-client-sdk` and `@launchdarkly/react-client-sdk`.
Add it to CI:

```yaml
# .github/workflows/flaglint.yml
- name: Enforce OpenFeature boundary
  run: npx flaglint@latest validate ./src --no-direct-launchdarkly
```

If your provider or bootstrap file legitimately imports the LaunchDarkly SDK, exclude it:

```bash
npx flaglint@latest validate ./src --no-direct-launchdarkly \
  --bootstrap-exclude "src/providers/**"
```

The [GitHub Actions enforcement guide](/blog/enforce-launchdarkly-migration-github-actions) covers
the full CI setup, including baseline mode for gradual rollout.

---

## Working through the migration plan

Run `flaglint migrate ./src --dry-run` to see the full migration plan across both server-side and
React call sites. Server-side calls with generated diffs are ready to apply with `--apply`. React
SDK hooks appear in the manual review section with file paths and flag keys.

A practical order:

1. Run `flaglint audit ./src` to get the full inventory and readiness score.
2. Set up `OpenFeatureProvider` at the app root with the LaunchDarkly web provider.
3. Convert `useVariation` call sites first — they have a near-one-to-one mapping.
4. Convert `useFlags` call sites — split each destructured key into a separate `useFlag` call.
5. Convert `useLDClient` call sites, handling any `track` calls separately.
6. Remove `LDProvider` / `withLDProvider` and uninstall the React SDK package.
7. Add `flaglint validate --no-direct-launchdarkly` to CI.

The [troubleshooting guide](/docs/guides/troubleshooting) covers common setup issues, including
the `--apply skips files` problem that can arise if your OpenFeature client binding is not
configured — relevant for any server-side call sites in the same migration batch.

---

The LaunchDarkly React SDK migration is the last structural step in removing front-end vendor
lock-in. FlagLint's scan and audit commands give you the complete call-site inventory; the
rewrites themselves are a series of small, localized changes in individual components. The boundary
enforcement in CI ensures the work doesn't drift back.
