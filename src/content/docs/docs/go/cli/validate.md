---
title: flaglint-go validate
description: Enforce a no-direct-LaunchDarkly policy, baseline-aware CI enforcement, and SARIF findings for Go.
lastUpdated: 2026-07-08
---

`flaglint-go validate` checks whether a Go codebase complies with a flag-usage policy. It's the only flaglint-go command that ever exits `1`.

## Options

| Flag | Description |
| --- | --- |
| `--no-direct-launchdarkly` | Fail if any direct LaunchDarkly Go SDK evaluation calls are found |
| `--bootstrap-exclude stringArray` | Glob pattern for files allowed to use the LaunchDarkly SDK directly (repeatable) |
| `-f, --format string` | Output format: `text` \| `sarif` (default `text`) |
| `-o, --output string` | Write report to a file instead of stdout |
| `--config string` | Path to config file |
| `--baseline string` | Baseline file for comparing against known debt |
| `--fail-on-new` | Exit `1` if any findings are not in the baseline |
| `--strict-types` | Additionally resolve findings only provable with real `go/types` information (interface satisfaction, transitive factory wrapping, cross-function method-value forwarding). Requires the module to build. See [Identity Model](/docs/go/concepts/identity-model/). |

## Blocking Policy Command

```bash
flaglint-go validate ./services --no-direct-launchdarkly
```

```text
✗ validate --no-direct-launchdarkly: 2 direct LaunchDarkly Go SDK call(s) found.

  checkout.go:13:16 — BoolVariation("checkout-v2")
  checkout.go:20:12 — IntVariation("discount-percentage")

These call sites must migrate to OpenFeature before this rule passes.
```

Pass output once no direct calls remain:

```text
✓ validate --no-direct-launchdarkly: no direct LaunchDarkly Go SDK calls found.
  Scanned 2 file(s).
```

## Baseline Mode

`--baseline`/`--fail-on-new` runs **independently** of `--no-direct-launchdarkly` — use it alone to adopt CI enforcement before existing debt is resolved:

```bash
flaglint-go audit ./services --write-baseline .flaglint-baseline.json
flaglint-go validate ./services --baseline .flaglint-baseline.json --fail-on-new
```

```text
Scanned 2 file(s). Found 2 LaunchDarkly Go SDK usage(s).
✓ No new findings beyond baseline
```

Once new debt is introduced:

```text
Scanned 2 file(s). Found 3 LaunchDarkly Go SDK usage(s).
Error: 1 new finding(s) not in baseline:
  - launchdarkly:BoolVariation:new-feature-flag:checkout.go
```

Re-run `--write-baseline` whenever you accept new debt on purpose.

A baseline also tracks each finding's *occurrence count*, not just which fingerprints are known — so a brand-new **duplicate** of an already-baselined call (copy-pasted into the same file) is caught too, even though its fingerprint is already in the baseline:

```text
Scanned 2 file(s). Found 3 LaunchDarkly Go SDK usage(s).
Error: 1 new finding(s) not in baseline:
  - launchdarkly:BoolVariation:checkout-v2:checkout.go
```

A baseline written by an older version of flaglint-go (or by flaglint-js, which doesn't have this extension yet) works the same as always — falling back to plain fingerprint-set comparison when a baseline has no occurrence counts recorded.

## SARIF

```bash
flaglint-go validate ./services --no-direct-launchdarkly --format sarif --output flaglint.sarif
```

SARIF findings use rule ID `flaglint.go.direct-launchdarkly` (Go-namespaced — distinct from flaglint-js's `flaglint.direct-launchdarkly`, per the [cross-tool contract](https://github.com/flaglint/flaglint-go/blob/main/docs/adr/003-cross-tool-contract.md)). Note: unlike flaglint-js's 0-based ESTree columns, Go's `token.Position.Column` is already 1-based, so flaglint-go does not add 1 to column numbers.

## Bootstrap Exclusions

Use `--bootstrap-exclude` for files allowed to use the SDK directly (a provider-wiring package, for example):

```bash
flaglint-go validate ./services --no-direct-launchdarkly --bootstrap-exclude "internal/openfeature-bootstrap/**"
```

## Exit Codes

| Code | Meaning |
| --- | --- |
| 0 | Success — no policy violations |
| 1 | Policy failure — direct calls found (`--no-direct-launchdarkly`) or new findings beyond baseline (`--fail-on-new`) |
| 2 | Invalid input — bad directory, bad `--format`, malformed config/baseline |
| 3 | Internal error |
| 130 | Interrupted (Ctrl-C) |

`scan` and `audit` never exit anything but `0` (barring a tool error) — `validate` is the only policy gate.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/cli/validate.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [Identity Model](/docs/go/concepts/identity-model/)
