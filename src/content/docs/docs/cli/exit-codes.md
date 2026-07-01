---
title: Exit Codes
description: Stable exit code contract for FlagLint v1.x — 0=success, 1=policy failure, 2=invalid input, 3=internal error, 130=interrupted.
lastUpdated: 2026-07-01
---

FlagLint uses a stable, machine-readable exit code contract. Codes `0`, `1`, `2`, and `130` are
guaranteed stable across all v1.x releases and safe to use in CI pipelines and scripts.

## Exit Code Table

| Code | Meaning | When it occurs |
|------|---------|----------------|
| `0` | Success — no blocking failures | Scan or audit completed; no policy violations found; no new findings beyond baseline |
| `1` | Policy or validation failure | `validate --no-direct-launchdarkly` found direct LD calls; new findings beyond baseline (`--fail-on-new`); dirty working tree without `--allow-dirty` |
| `2` | Invalid input | Bad `--format` value; directory not found or not a directory; missing or malformed baseline file; unsupported provider in config |
| `3` | Internal FlagLint error | Unexpected runtime exception in FlagLint itself — please [report as a bug](https://github.com/flaglint/flaglint/issues/new) |
| `130` | Interrupted (SIGINT) | User pressed Ctrl-C |

## Per-Command Notes

| Command | Exit `0` | Exit `1` | Exit `2` |
|---------|----------|----------|----------|
| `scan` | Always on completion (scan is inventory, not enforcement) | — | Invalid `--format`; directory not found; bad config |
| `audit` | Always on completion (audit is informational only) | — | Invalid `--format`; directory not found; bad config |
| `migrate` (default) | Plan written successfully | — | Directory not found; bad config |
| `migrate --dry-run` | Diffs printed | — | Directory not found |
| `migrate --apply` | Transformations applied | Dirty working tree (without `--allow-dirty`) | Directory not found |
| `validate` | No policy violations | Direct LD calls found; new findings beyond baseline | Invalid `--format`; bad baseline; bad config |

## Why exit 1 does not mean the tool crashed

Exit code `1` means **FlagLint ran successfully and found a policy violation** — not that the
tool failed. This distinction matters in CI logs:

- `exit 0` → Clean. No violations found.
- `exit 1` → FlagLint worked correctly; your code has a flag debt issue to address.
- `exit 2` → You passed a bad argument or your config is invalid; fix the command invocation.
- `exit 3` → FlagLint itself hit an unexpected error; please [file a bug report](https://github.com/flaglint/flaglint/issues/new).
- `exit 130` → Interrupted by the user (Ctrl-C).

This matches the POSIX convention used by linters such as ESLint (`exit 1` = lint errors found)
and allows CI steps to distinguish tool failure from policy failure without parsing stdout.

## CI YAML example

The following GitHub Actions workflow runs `flaglint validate` as a policy gate. It expects
`exit 0` (clean) or `exit 1` (violations found), and treats any other exit code as a
misconfiguration or internal error.

```yaml
name: FlagLint Policy Gate

on:
  pull_request:
  push:
    branches: [main]

jobs:
  flaglint-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "22"

      # Exit 0 → no direct LD calls (clean).
      # Exit 1 → direct LD calls found (policy violation — fail the build).
      # Exit 2 → bad --format or config error (fix the workflow invocation).
      - name: Validate no direct LaunchDarkly calls
        run: npx flaglint@latest validate ./src --no-direct-launchdarkly

      # Optional: emit SARIF for GitHub Code Scanning annotations
      - name: Validate (SARIF output)
        if: always()
        run: npx flaglint@latest validate ./src --no-direct-launchdarkly --format sarif --output flaglint.sarif
        continue-on-error: true

      - name: Upload SARIF to GitHub Code Scanning
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: flaglint.sarif
```

See the [GitHub Actions integration guide](/docs/integrations/github-actions/) and
the [`validate` command reference](/docs/cli/validate/) for the full set of options
including `--bootstrap-exclude` for allowing provider setup files.

## Stability guarantee

Codes `0`, `1`, `2`, and `130` are stable across all FlagLint v1.x releases.
Code `3` (internal error) may be refined in future releases but will always indicate
an unexpected tool failure (not a policy result).

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/cli/exit-codes.md)
- [Report an issue](https://github.com/flaglint/flaglint/issues/new)
- Next: [Express Guide](/docs/guides/express/)
