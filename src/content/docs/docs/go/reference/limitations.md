---
title: flaglint-go Limitations
description: Current boundaries and non-goals of flaglint-go.
lastUpdated: 2026-07-06
---

flaglint-go is intentionally narrow and honest about what it can't yet prove.

## No Automated Migration

There is no `migrate`/`--apply` command. flaglint-go audits and enforces policy; it does not rewrite source. If you need automated OpenFeature rewrites today, that capability currently exists only in [flaglint-js](/docs/quickstart/) for Node.js/TypeScript.

## Identity Resolution Gaps

flaglint-go proves client identity from syntax alone — no build, no `go/types`. That means a few real patterns aren't resolved yet, all filed as tracked, documented gaps rather than silently guessed at:

- **Chained factory-call-then-method** (`pkg.GetLdClient().Method(...)` with no intermediate variable) — [issue #20](https://github.com/flaglint/flaglint-go/issues/20)
- **Method values** (`f := client.BoolVariation; f(...)`) — [issue #6](https://github.com/flaglint/flaglint-go/issues/6)
- **Interface satisfaction** (a client known only through an interface type) — [issue #15](https://github.com/flaglint/flaglint-go/issues/15)
- **Block-scoped variable shadowing** within a single function — [issue #5](https://github.com/flaglint/flaglint-go/issues/5). Unlike every other gap below, this one **can cause a false positive**: a variable re-`:=`'d to an unrelated value inside a nested block is still treated as the outer real client.
- **A factory function returning a wrapper type**, not `*ld.LDClient` itself — [issue #16](https://github.com/flaglint/flaglint-go/issues/16)
- **Nested `go.mod` files** within one scanned tree (monorepo submodules) — [issue #17](https://github.com/flaglint/flaglint-go/issues/17)

Every gap above fails safe (a missed detection, never a false positive) except block-scoped shadowing, noted above. See [Identity Model](/docs/go/concepts/identity-model/) and [ADR 004](https://github.com/flaglint/flaglint-go/blob/main/docs/adr/004-whole-scan-identity-resolution.md) for the full design and reasoning behind each.

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
