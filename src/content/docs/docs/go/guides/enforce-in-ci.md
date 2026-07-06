---
title: Enforce a No-New-LaunchDarkly Policy for Go in CI
description: Use flaglint-go validate in GitHub Actions to block new direct LaunchDarkly Go SDK calls, adopting gradually via baseline mode.
lastUpdated: 2026-07-06
---

flaglint-go has no published GitHub Action (unlike flaglint-js's `flaglint/flaglint` composite action) — install it in your workflow with `go install` or Homebrew, then run `validate` directly. Both are one extra step.

## Install in CI

**Via `go install`** (needs `actions/setup-go`, already common in Go CI):

```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.23'
- run: go install github.com/flaglint/flaglint-go/cmd/flaglint-go@latest
```

**Via Homebrew** (no Go toolchain needed — `brew` is preinstalled on GitHub-hosted `ubuntu-latest` and `macos-latest` runners):

```yaml
- run: brew install flaglint/tap/flaglint-go
```

## Recommended: Adopt With a Baseline First

Turning on `--no-direct-launchdarkly` immediately fails CI for every existing call site — usually not what you want on day one. Capture current debt as a baseline once, commit it, and fail only on *new* debt going forward:

```bash
# Run once, locally, and commit the result
flaglint-go audit ./services --write-baseline .flaglint-baseline.json
```

```yaml
name: FlagLint Go
on: [pull_request]

jobs:
  flaglint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
      - run: go install github.com/flaglint/flaglint-go/cmd/flaglint-go@latest
      - run: flaglint-go validate ./services --baseline .flaglint-baseline.json --fail-on-new
```

This fails the build only when a call site's fingerprint isn't already in `.flaglint-baseline.json`. Re-run `--write-baseline` (and commit the update) whenever you accept new debt on purpose or resolve existing findings.

## Blocking All Direct Calls

Once existing debt is cleared (or for a codebase starting clean), enforce the strict policy instead:

```yaml
- run: flaglint-go validate ./services --no-direct-launchdarkly
```

Do not add `continue-on-error: true` to this step — the job should fail when violations exist.

### Bootstrap Exclusions

Files that legitimately wire the LaunchDarkly client directly (a provider-setup package, for example) can be exempted:

```yaml
- run: |
    flaglint-go validate ./services \
      --no-direct-launchdarkly \
      --bootstrap-exclude "internal/openfeature-bootstrap/**"
```

## SARIF Upload for GitHub Code Scanning

```yaml
name: FlagLint Go (SARIF)
on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
      - run: go install github.com/flaglint/flaglint-go/cmd/flaglint-go@latest

      - name: Validate no direct LaunchDarkly Go SDK calls
        run: |
          flaglint-go validate ./services \
            --no-direct-launchdarkly \
            --format sarif \
            --output flaglint-go-validation.sarif

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: flaglint-go-validation.sarif
```

`if: always()` belongs on the upload step, not the validate step, so GitHub Code Scanning still ingests results even when validation fails. SARIF findings use rule ID `flaglint.go.direct-launchdarkly`.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/guides/enforce-in-ci.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- See also: [validate reference](/docs/go/cli/validate/)
- Next: [flaglint-go Overview](/docs/go/)
