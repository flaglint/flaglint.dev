---
title: "flaglint-go: We Field-Tested It Against Real Repos and Found Zero Recall"
description: "flaglint-go is a native Go CLI for auditing LaunchDarkly Go SDK usage. Here's what happened when we ran it against real open-source Go repos, and how we fixed it without needing a type checker."
date: 2026-07-06
authors:
  - name: Krishan Sharma
    title: Founder and maintainer of FlagLint
    url: https://www.linkedin.com/in/krishansha/
tags: ["golang", "launchdarkly", "cli", "static-analysis", "open-source"]
---

flaglint-go is a native Go binary for auditing [LaunchDarkly Go server SDK](https://github.com/launchdarkly/go-server-sdk) usage — no Node.js required.

```bash
brew install flaglint/tap/flaglint-go
flaglint-go audit ./services
```

It's the Go-native counterpart to flaglint-js, and it shares the same non-negotiable rule: a variable is only ever treated as a LaunchDarkly client when its identity can be *proven* — never by matching a method name in isolation. That rule is why flaglint-js earned trust in the first place, and flaglint-go inherited it from day one.

<!-- Image 1 (intro): "gopher" or Go-related terminal image
     Suggested source: unsplash.com search "terminal code" or a Go gopher illustration
     Alt: A terminal running a Go CLI tool -->

## The problem we didn't expect

Before shipping, we validated flaglint-go the way you'd hope any static analysis tool gets validated: not just against synthetic fixtures we wrote ourselves, but against real, unmodified, open-source Go repositories known to use the LaunchDarkly SDK — including the [official `launchdarkly-labs/ld-sample-app-go`](https://github.com/launchdarkly-labs/ld-sample-app-go), plus weaviate/weaviate, CMS-Enterprise/mint-app, and e2b-dev/infra.

The result: **zero false positives, but also zero recall on every single repo with genuine usage.**

That's a striking failure. Not "missed a few edge cases" — missed *all of it*, on the official sample app included. The scanner wasn't broken; it was doing exactly what it was designed to do (prove identity syntactically, never guess by name) — it just turned out real Go code almost never wires up the LaunchDarkly client the simple way our synthetic fixtures assumed.

## What real code actually does

Three different repos, three different indirection patterns, none of them exotic:

**The official sample app** wires the client through a package-level singleton getter, called from a different package entirely:

```go
// package ldclient
func GetLdClient() *ld.LDClient { /* ... */ }

// package api, a different file, a different package
client := ldclient.GetLdClient()
client.BoolVariation("test-flag", ctx, false)
```

**weaviate** stores the client into a wrapper struct via composite literal, then reaches it through a two-level field chain — on a *generic* struct:

```go
type LDIntegration struct {
    ldClient *ld.LDClient
}

type FeatureFlag[T SupportedTypes] struct {
    ldInteg *LDIntegration
}

// inside one of FeatureFlag[T]'s methods:
flag, err := f.ldInteg.ldClient.StringVariation(f.key, f.ldInteg.ldContext, v)
```

**mint-app** passes an already-constructed client into a struct's constructor as a plain function parameter — there's no assignment to trace at all; the parameter's declared type is the only place identity is ever established.

None of these require real type-checking to resolve. In every case, the proof of identity is available directly from the AST — a struct's declared field type, a function's declared parameter or return type — the same "trust the syntax, no build required" spirit as the rest of the scanner. What they *do* require is looking at more than one file at a time, since the code that constructs the client and the code that uses it are routinely in a different file, sometimes a different package, from wherever the binding was first established.

<!-- Image 2 (middle): "black screen with code" or similar dark terminal/code screenshot
     Alt: Source code showing struct definitions and method calls -->

## Rebuilding around a whole-scan pass

We rearchitected the scanner from a per-file model (read, parse, detect, discard — one file at a time) into a whole-scan pass: parse every file up front, then resolve identity across the entire scan before any detection runs. That closed all three gaps:

- **Composite-literal struct-field binding** — `&LDIntegration{ldClient: client}` binds the field when `client` is itself already bound.
- **Multi-level field-selector chains**, including through generics — a struct's fields can be declared in a different file than where they're used.
- **Cross-package factory/getter functions**, resolved via real `go.mod`-derived import paths — never a name-based guess. (We explicitly considered and rejected matching by "last segment of the import path" as a shortcut — that's a name heuristic wearing an import-path costume, exactly the kind of thing our non-negotiable identity rule exists to prevent.)
- **Parameter-typed client bindings** — a parameter declared `*ld.LDClient` is bound from its type alone, no assignment to trace.

We also found and fixed a bug that had nothing to do with the original plan: Go generics. weaviate's `FeatureFlag[T]` broke a piece of code that had only ever been tested against non-generic structs — a method receiver on a generic type has a different AST shape (`*ast.IndexExpr`) than a plain one, and the scanner silently failed to resolve it at all. Found only by testing against real code that happened to use generics.

## The bug our own review process caught

Before merging any of this, we ran an independent review pass — a fresh reviewer with no context on the implementation, adversarially checking the diff. It found something real: two of the new whole-scan indices (struct field types, and package-level/struct-field bindings) were keyed by bare name across the *entire* scan, not scoped to a package. Go allows two completely unrelated packages to each declare their own `Service` struct with their own `Client` field — and a genuinely-bound client in one package would have incorrectly matched a same-named, unconnected field in a totally different package.

That's exactly the class of false positive our whole identity model exists to prevent, and it slipped through the first pass. We fixed it by partitioning every whole-scan index by package — matching how an unqualified identifier is only ever visible within its own package in real Go anyway — added regression fixtures reproducing the exact collision, and re-verified against all three real repos to confirm detection was unaffected by the fix.

## Verifying against the real repos, again

After the fix, we re-cloned and re-scanned the same repos:

```text
$ flaglint-go audit ./ld-sample-app-go
Scan complete — 1 unique flag(s) across 1 call site(s) (3 file(s))
Migration readiness: 100/100 · ready
  1 low risk · 0 medium risk · 0 high risk
```

```text
$ flaglint-go audit ./weaviate
Scan complete — 0 unique flag(s) across 4 call site(s) (4512 file(s))
Migration readiness: 0/100 · complex
  0 low risk · 0 medium risk · 4 high risk
```

weaviate's four call sites all show as high-risk *dynamic* keys — correctly so, since the flag key there (`f.key`) is a runtime struct field, not a string literal. That's the honest answer, not a false claim of full static resolution.

## What's still not covered

We're not going to pretend this closes every gap. e2b-dev/infra's usage is still undetected — it takes a bound client's method value and passes it through a generic helper function, which is a genuinely harder problem (interprocedural data-flow, not just "look at more files"). That, along with a handful of narrower gaps found along the way, is filed as tracked, public issues rather than silently swept under the rug — see the [Supported Scope](/docs/go/reference/supported-scope/) and [Limitations](/docs/go/reference/limitations/) pages for the full, honest list.

## Get started

```bash
brew install flaglint/tap/flaglint-go
flaglint-go audit ./services
```

- [Quickstart](/docs/go/quickstart/)
- [Identity Model](/docs/go/concepts/identity-model/) — the full design behind all of the above
- [Enforce in CI](/docs/go/guides/enforce-in-ci/)

<!-- Image 3 (end): "macbook pro on brown wooden table" or similar closing dev-workstation image
     Alt: A developer's workstation, representing ongoing CLI tool development -->
