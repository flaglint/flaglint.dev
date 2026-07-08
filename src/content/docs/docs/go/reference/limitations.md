---
title: flaglint-go Limitations
description: Current boundaries and non-goals of flaglint-go.
lastUpdated: 2026-07-08
---

flaglint-go is intentionally narrow and honest about what it can't yet prove.

## No Automated Migration

There is no `migrate`/`--apply` command. flaglint-go audits and enforces policy; it does not rewrite source. If you need automated OpenFeature rewrites today, that capability currently exists only in [flaglint-js](/docs/quickstart/) for Node.js/TypeScript.

## Identity Resolution Gaps

flaglint-go proves client identity in two layers: Phase 1 (the default `audit`/`scan`/`validate` behavior) resolves purely from syntax — no build, no `go/types`. An opt-in `--strict-types` pass (Phase 2) additionally loads the target module with real `go/types` information to resolve a small number of patterns pure syntax structurally can't — see [Identity Model](/docs/go/concepts/identity-model/) for the split and why it's opt-in (it requires the module to build).

As of 2026-07-08, there are no identity-resolution gaps currently tracked against flaglint-go — every gap previously documented on this page has been resolved. New gaps are filed as found; see the [issue tracker](https://github.com/flaglint/flaglint-go/issues) for the current list.

Recently closed, kept here for continuity:

- **Chained factory-call-then-method** (`pkg.GetLdClient().Method(...)` with no intermediate variable) — [issue #20](https://github.com/flaglint/flaglint-go/issues/20), resolved in Phase 1.
- **Method values within one function** (`f := client.BoolVariation; f(...)`) — [issue #6](https://github.com/flaglint/flaglint-go/issues/6), resolved in Phase 1. The harder cross-function case (a method value passed as an argument into a different function) — [issue #26](https://github.com/flaglint/flaglint-go/issues/26) — is resolved under `--strict-types`.
- **Interface satisfaction** (a client known only through an interface type) — [issue #15](https://github.com/flaglint/flaglint-go/issues/15), resolved under `--strict-types`.
- **Block-scoped variable shadowing** within a single function — [issue #5](https://github.com/flaglint/flaglint-go/issues/5), resolved in Phase 1. This was the one gap that could cause a false positive rather than a missed detection.
- **A factory function returning a wrapper type**, not `*ld.LDClient` itself — [issue #16](https://github.com/flaglint/flaglint-go/issues/16), resolved under `--strict-types`.
- **Nested `go.mod` files** within one scanned tree (monorepo submodules) — [issue #17](https://github.com/flaglint/flaglint-go/issues/17), resolved in Phase 1.
- **A struct field declared `*ld.LDClient` with no observed construction anywhere in the scanned tree** (the dominant Go dependency-injection pattern) — [issue #32](https://github.com/flaglint/flaglint-go/issues/32), resolved in Phase 1.
- **A composite literal directly initializing a package-level `var`** (`var svc = &Svc{Client: svcClient}`, never inside any function body) — [issue #33](https://github.com/flaglint/flaglint-go/issues/33), resolved in Phase 1.

Every gap above failed safe (a missed detection, never a false positive) except block-scoped shadowing, noted above. See [Identity Model](/docs/go/concepts/identity-model/) and [ADR 004](https://github.com/flaglint/flaglint-go/blob/main/docs/adr/004-whole-scan-identity-resolution.md) for the full design and reasoning behind each.

## Not Production Staleness Analysis

flaglint-go does not query LaunchDarkly and does not identify production-stale flags. It cannot know flag age, owner, evaluation history, environment configuration, or production usage from source alone.

## Outside Detection Coverage

- Browser/mobile LaunchDarkly SDKs.
- Non-LaunchDarkly feature-flag providers.
- Runtime-only flag key construction that can't be resolved statically (reported as high-risk "dynamic", not silently ignored).

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/reference/limitations.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [flaglint-go Overview](/docs/go/)
