---
title: flaglint-go Quickstart
description: Install flaglint-go, run your first audit, and enforce a CI policy — no Node.js required.
lastUpdated: 2026-07-06
---

## Installation

**Homebrew (macOS / Linux)**
```bash
brew install flaglint/tap/flaglint-go
```

**go install**
```bash
go install github.com/flaglint/flaglint-go/cmd/flaglint-go@latest
```

Prebuilt release binaries are also available from the [releases page](https://github.com/flaglint/flaglint-go/releases).

## Requirements

- A Go project using `github.com/launchdarkly/go-server-sdk` v6 or v7.
- Nothing else — flaglint-go is a single static binary with no runtime dependencies. It does not need your project to build; it parses source with `go/parser` only.

## 1. Run an Audit

```bash
flaglint-go audit ./services
```

Example output from a small service wiring the client through a package-level singleton getter (a pattern flaglint-go resolves across files — see [Identity Model](/docs/go/concepts/identity-model/)):

```go
// pkg/flags/client.go
var client *ld.LDClient

func Client() *ld.LDClient { return client }
```

```go
// checkout.go
client := flags.Client()
enabled, _ := client.BoolVariation("checkout-v2", ctx, false)
pct, _ := client.IntVariation("discount-percentage", ctx, 0)
```

```text
Scan complete — 2 unique flag(s) across 2 call site(s) (0ms, 2 file(s))
Migration readiness: 100/100 · ready
  2 low risk · 0 medium risk · 0 high risk
```

**The three commands and when to use each:**

- `audit` — risk-ranked overview with a migration-readiness score. Start here.
- `scan` — structured, file-level inventory (JSON/Markdown) for automation or deeper review.
- `validate` — CI gate that blocks new direct LaunchDarkly evaluation calls.

## 2. Inspect Detailed Inventory With Scan

```bash
flaglint-go scan ./services --format json
```

```json
{
  "scannedFiles": 2,
  "totalUsages": 2,
  "uniqueFlags": ["checkout-v2", "discount-percentage"],
  "usages": [
    {
      "flagKey": "checkout-v2",
      "isDynamic": false,
      "file": "checkout.go",
      "line": 13,
      "callType": "BoolVariation",
      "fingerprint": "launchdarkly:BoolVariation:checkout-v2:checkout.go",
      "risk": "low"
    }
  ]
}
```

## 3. Enforce a Policy in CI

```bash
flaglint-go validate ./services --no-direct-launchdarkly
```

```text
✗ validate --no-direct-launchdarkly: 2 direct LaunchDarkly Go SDK call(s) found.

  checkout.go:13:16 — BoolVariation("checkout-v2")
  checkout.go:20:12 — IntVariation("discount-percentage")

These call sites must migrate to OpenFeature before this rule passes.
```

`validate` is the only command that ever exits `1` — `scan` and `audit` always exit `0` unless something breaks (a bad directory, malformed config).

## 4. Adopt Gradually With a Baseline

Turning on `--no-direct-launchdarkly` immediately fails CI for every existing call site. To adopt flaglint-go without a big-bang migration, capture current debt as a baseline and only fail on *new* debt:

```bash
# Capture current findings once, commit the file
flaglint-go audit ./services --write-baseline .flaglint-baseline.json

# In CI: pass as long as no new debt was introduced
flaglint-go validate ./services --baseline .flaglint-baseline.json --fail-on-new
```

```text
Scanned 2 file(s). Found 2 LaunchDarkly Go SDK usage(s).
✓ No new findings beyond baseline
```

`--baseline`/`--fail-on-new` runs independently of `--no-direct-launchdarkly` — most teams adopt with baseline mode alone, then turn on the strict policy later once existing debt is cleared.

---

**Further reading:** [CLI reference](/docs/go/cli/audit/) · [Identity Model](/docs/go/concepts/identity-model/) · [Enforce in CI guide](/docs/go/guides/enforce-in-ci/)

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/quickstart.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [CLI Reference](/docs/go/cli/audit/)
