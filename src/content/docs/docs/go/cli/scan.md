---
title: flaglint-go scan
description: Structured, file-level LaunchDarkly Go SDK usage inventory. Always exits 0.
lastUpdated: 2026-07-08
---

`flaglint-go scan` walks a directory and reports every detected LaunchDarkly Go SDK call site as a structured inventory — no risk scoring, no readiness score, just the raw findings. Use `audit` instead when you want a risk-ranked summary.

## Command

```bash
flaglint-go scan [dir] [flags]
```

## Options

| Flag | Description |
| --- | --- |
| `-f, --format string` | Output format: `json` \| `markdown` (default `markdown`) |
| `-o, --output string` | Write report to a file instead of stdout |
| `--config string` | Path to config file |
| `--strict-types` | Additionally resolve findings only provable with real `go/types` information (interface satisfaction, transitive factory wrapping, cross-function method-value forwarding). Requires the module to build. See [Identity Model](/docs/go/concepts/identity-model/). |

## Example Output (Markdown)

```text
# FlagLint Go Scan Report

Scanned 2 file(s) in 0ms — 2 unique flag(s) across 2 call site(s).

## Flags

| Flag | Call Type | Risk | File |
|---|---|---|---|
| `checkout-v2` | BoolVariation | low | checkout.go:13 |
| `discount-percentage` | IntVariation | low | checkout.go:20 |
```

## Example Output (JSON)

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
      "column": 16,
      "callType": "BoolVariation",
      "fingerprint": "launchdarkly:BoolVariation:checkout-v2:checkout.go",
      "stalenessSignals": [],
      "language": "go",
      "sdk": "go-server-sdk-v7",
      "risk": "low"
    }
  ],
  "warnings": []
}
```

Fingerprints (`launchdarkly:callType:flagKey:normalizedPath[:dynamicIndex]`) are stable across line-number changes and match flaglint-js's format byte-for-byte — see the [cross-tool contract](https://github.com/flaglint/flaglint-go/blob/main/docs/adr/003-cross-tool-contract.md).

## Exit Behavior

`scan` always exits `0` unless the tool itself errors (invalid directory, bad `--format`, unreadable config). It never fails based on what it finds — use `flaglint-go validate` for a CI gate.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/cli/scan.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [audit](/docs/go/cli/audit/)
