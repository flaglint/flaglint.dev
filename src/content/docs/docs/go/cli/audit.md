---
title: flaglint-go audit
description: Risk-ranked LaunchDarkly Go SDK usage report with a migration-readiness score.
lastUpdated: 2026-07-06
---

`flaglint-go audit` scans your source and classifies every detected LaunchDarkly Go SDK call by risk level, producing a migration-readiness score. It never modifies files.

## Command

```bash
flaglint-go audit [dir] [flags]
```

## Options

| Flag | Description |
| --- | --- |
| `-f, --format string` | Output format: `json` \| `markdown` (default `markdown`) |
| `-o, --output string` | Write report to a file instead of stdout |
| `--config string` | Path to config file |
| `--write-baseline string` | Write current finding fingerprints to a baseline file for use with `validate --baseline --fail-on-new` |

## Risk Levels

| Method | Risk | Notes |
|---|---|---|
| `BoolVariation` / `StringVariation` / `IntVariation` / `Float64Variation` (and their `*Ctx` forms) | Low | Static key only |
| `JSONVariation` (and `*Ctx`) | Medium | Pointer output, manual review |
| `*VariationDetail(Ctx)` methods | High | Returns `EvaluationDetail` — no direct OpenFeature equivalent |
| `AllFlagsState` | High | Bulk call, no single-flag equivalent |
| Dynamic flag key (identifier, `fmt.Sprintf`, string concatenation) | High | Cannot resolve statically — overrides the method's own risk level |

## Example Output

```text
Scan complete — 2 unique flag(s) across 2 call site(s) (0ms, 2 file(s))
Migration readiness: 100/100 · ready
  2 low risk · 0 medium risk · 0 high risk
```

A codebase with genuine high-risk usage:

```text
Scan complete — 0 unique flag(s) across 4 call site(s) (2.3s, 4512 file(s))
Migration readiness: 0/100 · complex
  0 low risk · 0 medium risk · 4 high risk
```

(All four call sites use a dynamic, per-instance flag key — correctly classified high risk regardless of the underlying method, per the dynamic-key rule above.)

## Migration Readiness

The score is the ratio of low/medium-risk (safely reviewable) call sites to total detected call sites, expressed 0–100. `N/A` is reported when zero direct calls are found — there's nothing to score.

## Baseline Mode (`--write-baseline`)

```bash
flaglint-go audit ./services --write-baseline .flaglint-baseline.json
```

```json
{
  "version": "1",
  "createdAt": "2026-07-06T19:23:09Z",
  "flaglintVersion": "dev",
  "fingerprints": [
    "launchdarkly:BoolVariation:checkout-v2:checkout.go",
    "launchdarkly:IntVariation:discount-percentage:checkout.go"
  ]
}
```

Commit this file to source control, then use it with `flaglint-go validate --baseline <file> --fail-on-new` to adopt CI enforcement without requiring all existing debt to be fixed first. See [Quickstart: Adopt Gradually With a Baseline](/docs/go/quickstart/#4-adopt-gradually-with-a-baseline).

## Exit Behavior

`audit` always exits `0` unless the tool itself errors. It is informational — use `flaglint-go validate --no-direct-launchdarkly` to fail a build.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/cli/audit.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [validate](/docs/go/cli/validate/)
