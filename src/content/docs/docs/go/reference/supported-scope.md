---
title: flaglint-go Supported Scope
description: What flaglint-go detects, reports, and excludes.
lastUpdated: 2026-07-08
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

flaglint-go proves client identity in two layers. Phase 1 (the default `audit`/`scan`/`validate` behavior, always on) resolves purely from syntax — no `go/types`, no build required. An opt-in `--strict-types` pass (Phase 2) additionally resolves a small number of patterns pure syntax structurally can't. See [Identity Model](/docs/go/concepts/identity-model/) for the full split and detail on each of these.

### Phase 1 — default, no build required

| Pattern | Resolved |
| --- | --- |
| Direct constructor binding (`x := ld.MakeClient(...)`) | Yes |
| Package-level `var` and struct-field assignment | Yes — across the whole scan, not just one file |
| Composite-literal struct-field binding (`&Integration{ldClient: client}`) | Yes — including a literal that directly initializes a package-level `var`, never inside any function body |
| Multi-level field-selector chains (`f.integ.ldClient.Method(...)`), including generic structs | Yes |
| Cross-package factory/getter functions (`pkg.GetLdClient()`) | Yes — requires a `go.mod` to compute real import paths |
| Parameter-typed client bindings (`func f(client *ld.LDClient)`) | Yes |
| A struct field declared `*ld.LDClient`, with no observed construction anywhere in the scanned tree | Yes — the field's declared type alone is sufficient proof, the dominant Go dependency-injection pattern |
| Chained factory-call-then-method (`pkg.GetLdClient().Method(...)`, no intermediate variable) | Yes |
| Method values within one function (`f := client.BoolVariation; f(...)`) | Yes |
| Block-scoped variable shadowing within one function | Yes — no longer a false-positive risk |
| Nested `go.mod` files within one scanned tree (monorepo submodules) | Yes |

### Phase 2 — `--strict-types` (opt-in, requires the module to build)

| Pattern | Resolved |
| --- | --- |
| Interface satisfaction (client known only through an interface type) | Yes |
| A factory function returning a wrapper type (not `*ld.LDClient` itself) | Yes |
| A method value passed as an argument into a *different* function (the "forwarding function" pattern) | Yes |

Every pattern above fails safe when it can't be resolved (a missed detection, never a false positive) — flaglint-go's non-negotiable rule is to under-detect rather than guess. See [Limitations](/docs/go/reference/limitations/) for anything currently outside detection coverage.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/reference/supported-scope.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [Limitations](/docs/go/reference/limitations/)
