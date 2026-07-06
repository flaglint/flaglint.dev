---
title: flaglint-go Supported Scope
description: What flaglint-go detects, reports, and excludes.
lastUpdated: 2026-07-06
---

This page covers **flaglint-go**, the Go CLI. For the JavaScript/TypeScript CLI's scope, see [flaglint-js: Supported Scope](/docs/reference/supported-scope/).

flaglint-go detects LaunchDarkly Go server-side SDK evaluation calls from:

- `github.com/launchdarkly/go-server-sdk/v6`
- `github.com/launchdarkly/go-server-sdk/v7`

## API Coverage

| Method | Detected | Risk |
| --- | --- | --- |
| `BoolVariation` / `StringVariation` / `IntVariation` / `Float64Variation` | Yes | Low (static key) |
| `*Ctx` forms of the above | Yes | Low (static key) |
| `JSONVariation` (and `*Ctx`) | Yes | Medium |
| `*VariationDetail(Ctx)` methods | Yes | High |
| `AllFlagsState` | Yes | High |
| Dynamic flag keys | Yes | High (overrides the method's own risk) |
| Browser/mobile LaunchDarkly SDKs | No | — |
| Non-LaunchDarkly providers | No | — |

## Identity Resolution Coverage

flaglint-go proves client identity syntactically (no `go/types`, no build required) across an entire scan. See [Identity Model](/docs/go/concepts/identity-model/) for detail on each of these:

| Pattern | Resolved |
| --- | --- |
| Direct constructor binding (`x := ld.MakeClient(...)`) | Yes |
| Package-level `var` and struct-field assignment | Yes — across the whole scan, not just one file |
| Composite-literal struct-field binding (`&Integration{ldClient: client}`) | Yes |
| Multi-level field-selector chains (`f.integ.ldClient.Method(...)`), including generic structs | Yes |
| Cross-package factory/getter functions (`pkg.GetLdClient()`) | Yes — requires a `go.mod` to compute real import paths |
| Parameter-typed client bindings (`func f(client *ld.LDClient)`) | Yes |
| Chained factory-call-then-method (`pkg.GetLdClient().Method(...)`, no intermediate variable) | **No** — [tracked](https://github.com/flaglint/flaglint-go/issues/20) |
| Method values (`f := client.BoolVariation; f(...)`) | No — [tracked](https://github.com/flaglint/flaglint-go/issues/6) |
| Interface satisfaction (client known only through an interface type) | No — [tracked](https://github.com/flaglint/flaglint-go/issues/15) |
| Block-scoped variable shadowing within one function | No — [tracked](https://github.com/flaglint/flaglint-go/issues/5) |
| A factory function returning a wrapper type (not `*ld.LDClient` itself) | No — [tracked](https://github.com/flaglint/flaglint-go/issues/16) |
| Nested `go.mod` files within one scanned tree (monorepo submodules) | Partial — [tracked](https://github.com/flaglint/flaglint-go/issues/17) |

Every "No" above is a documented false-negative risk, never a false-positive one — flaglint-go's non-negotiable rule is to under-detect rather than guess. See [Limitations](/docs/go/reference/limitations/).

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/reference/supported-scope.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [Limitations](/docs/go/reference/limitations/)
