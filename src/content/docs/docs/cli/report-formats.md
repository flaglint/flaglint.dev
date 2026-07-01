---
title: Report Formats
description: When to use JSON, Markdown, HTML, SARIF, and text output — comparison table, usage examples, and GitHub Code Scanning integration.
lastUpdated: 2026-07-01
---

FlagLint supports multiple output formats. Choose based on who consumes the output and how.

## Format comparison

| Format | Commands | Best for |
|--------|----------|----------|
| `markdown` | `scan`, `audit` | Human review, GitHub PR comments, Slack sharing |
| `json` | `scan`, `audit` | Automation, CI pipelines, custom dashboards, `jq` filtering |
| `html` | `scan`, `audit` | Sharing with non-technical stakeholders, archiving audit results |
| `sarif` | `scan`, `validate` | GitHub Code Scanning, IDE integrations, VS Code SARIF Viewer |
| `text` | `validate` | Terminal output, CI logs where markdown rendering is not available |

## When to use each format

**`markdown` (default for scan and audit)**

Readable in GitHub PR comments and any Markdown renderer. Use this when you want engineers
to read findings directly — in a PR review, a Slack post, or a shared Notion doc. This is the
default when `--format` is omitted.

**`json`**

Machine-readable structured output. Use with `jq` for filtering, in custom dashboards, or when
building integrations on top of FlagLint output. The full schema is documented at
[JSON Output Reference](/docs/reference/json-output/).

```bash
npx flaglint@latest scan ./src --format json | jq '.usages[] | select(.isStale == true)'
```

**`html`**

Self-contained single-file report. Use for sharing audit results with engineering managers or
stakeholders who do not have CLI access. The HTML file embeds all styles — no server needed,
just open in a browser.

```bash
npx flaglint@latest audit ./src --format html --output flaglint-audit.html
```

**`sarif`**

[SARIF 2.1.0](https://sarifweb.azurewebsites.net/) compliant output. Use with:
- **GitHub Code Scanning** — upload via `github/codeql-action/upload-sarif` to get inline PR annotations
- **VS Code SARIF Viewer** extension — view findings inline in your editor
- **Azure DevOps** SARIF integration

**`text`**

Plain-text output for `validate`. Use in CI logs where markdown rendering is not available and
you want a human-readable summary without table syntax.

## Usage examples

### Scan formats

```bash
npx flaglint@latest scan ./src --format json
npx flaglint@latest scan ./src --format markdown
npx flaglint@latest scan ./src --format html --output flaglint-inventory.html
npx flaglint@latest scan ./src --format sarif --output flaglint-inventory.sarif
```

### Validation SARIF

Use `validate --format sarif` for direct-SDK policy enforcement with GitHub Code Scanning:

```bash
npx flaglint@latest validate ./src \
  --no-direct-launchdarkly \
  --format sarif \
  --output flaglint-validation.sarif
```

### GitHub Code Scanning integration

Upload SARIF output to get inline annotations on pull requests:

```yaml
- name: Run FlagLint validate
  run: npx flaglint@latest validate ./src --no-direct-launchdarkly --format sarif --output flaglint.sarif
  continue-on-error: true

- name: Upload SARIF to GitHub Code Scanning
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: flaglint.sarif
```

Findings appear as inline annotations on the Files Changed tab of your pull request.

### Migration reports

```bash
npx flaglint@latest migrate ./src --output MIGRATION.md
npx flaglint@latest migrate ./src --dry-run
```

The default migration report summarizes evidence and migration readiness. `--dry-run` prints
reviewable diffs to stdout without writing any files.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/cli/report-formats.md)
- [Report an unsupported pattern](https://github.com/flaglint/flaglint/issues/new?template=unsupported_pattern.yml)
- Next: [Exit Codes](/docs/cli/exit-codes/)
