---
title: flaglint-go Overview
description: Audit LaunchDarkly Go server SDK usage with a native Go binary — no Node.js required.
lastUpdated: 2026-07-06
tableOfContents: false
---

flaglint-go is the Go-native counterpart to flaglint-js — a standalone binary for auditing [LaunchDarkly Go server SDK](https://github.com/launchdarkly/go-server-sdk) usage, built for teams that don't want a Node.js dependency in their toolchain.

```bash
brew install flaglint/tap/flaglint-go
flaglint-go audit ./services
```

No API key. No source upload. Runs locally against your checkout, using Go's own `go/parser`/`go/ast` — no build required, no `go/types` type-checking, no tree-sitter.

<div class="button-grid">
  <a href="/docs/go/quickstart">Quickstart</a>
  <a href="/docs/go/cli/audit">CLI Reference</a>
  <a href="/docs/go/concepts/identity-model">Identity Model</a>
  <a href="/docs/go/reference/supported-scope">Supported Scope</a>
</div>

## flaglint-go vs. flaglint-js

flaglint-go is an early-access, separate binary — not a Go mode bolted onto the npm CLI. They share the same fingerprint, exit-code, config, and SARIF conventions ([cross-tool contract](https://github.com/flaglint/flaglint-go/blob/main/docs/adr/003-cross-tool-contract.md)), but flaglint-go's command surface is currently narrower:

| | flaglint-js | flaglint-go |
|---|---|---|
| `scan` — structured inventory | ✅ | ✅ |
| `audit` — risk-ranked report + readiness score | ✅ | ✅ |
| `validate` — CI policy gate | ✅ | ✅ |
| `migrate` — automated OpenFeature rewrites | ✅ | ❌ not yet |
| `init` / `completion` | ✅ | ❌ not yet |

## What flaglint-go Does

- Performs local, syntax-only source analysis — no build, no `go/types`, no network access.
- Detects `github.com/launchdarkly/go-server-sdk` v6 and v7 evaluation calls, proven through import-alias tracing and constructor-call binding — never through name matching alone.
- Resolves common real-world indirection across an entire scan (not just one file): struct fields, composite literals, multi-level field chains, cross-package factory/getter functions, and typed parameters. See [Identity Model](/docs/go/concepts/identity-model/).
- Generates the same inventory/audit/validate reports as flaglint-js (JSON, Markdown, text, SARIF).

## What flaglint-go Does Not Do

- It does not rewrite code. There is no `migrate`/`--apply` command yet.
- It does not resolve every indirection pattern — method values, interface satisfaction, and a few other gaps are documented and tracked. See [Limitations](/docs/go/reference/limitations/).
- It does not query LaunchDarkly for flag age, owner, evaluation history, or production usage.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/index.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [Quickstart](/docs/go/quickstart/)
