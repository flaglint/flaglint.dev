---
title: Configuration
description: Configure scan scope, wrappers, and imported OpenFeature client bindings.
lastUpdated: 2026-07-01
---

FlagLint reads the first matching config file from:

```text
.flaglintrc
.flaglintrc.json
flaglint.config.json
```

Use `--config <path>` to pass a specific file.

## Example

```json
{
  "include": ["**/*.{ts,js}"],
  "exclude": ["**/node_modules/**", "**/dist/**"],
  "provider": "launchdarkly",
  "minFileCount": 0,
  "wrappers": ["evaluateFlag"],
  "openFeatureClientBindings": [
    {
      "importName": "openFeatureClient",
      "modulePatterns": ["**/platform/feature-flags"]
    }
  ]
}
```

## Fields

| Field | Default | Purpose |
| --- | --- | --- |
| `include` | `["**/*.{ts,tsx,js,jsx}"]` | Files to scan. |
| `exclude` | common build/test output | Files to skip. |
| `provider` | `launchdarkly` | Current implemented provider scope. |
| `minFileCount` | `0` | Optional local source review heuristic. |
| `wrappers` | `[]` | Wrapper functions to detect as flag evaluations. Accepts string names or object form (see below). |
| `openFeatureClientBindings` | `[]` | Imported shared client allowlist for `migrate --apply`. |
| `reportTitle` | unset | Optional report title. |
| `outputDir` | `.` | Directory for generated report files. |

## Wrappers — String and Object Form

The `wrappers` field accepts two forms, which can be mixed in the same array.

**String form** — any call to a function with that name is treated as a flag evaluation:

```json
{
  "wrappers": ["evaluateFlag", "getFeature"]
}
```

**Object form** — the call is only detected when the function is imported from a specific package, preventing false positives from unrelated functions with the same name:

```json
{
  "wrappers": [
    "evaluateFlag",
    {
      "import": "my-feature-sdk",
      "function": "getFlag",
      "flagKeyArgument": 0
    }
  ]
}
```

Object form fields:

| Field | Purpose |
| --- | --- |
| `import` | The package name that must be imported for this call to be detected. |
| `function` | The exported function name to detect. |
| `flagKeyArgument` | Zero-based index of the argument that holds the flag key. |

With the example above, `getFlag("my-flag-key", ctx)` is detected only when `getFlag` is imported from `my-feature-sdk`. A `getFlag` call imported from any other package is ignored.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint/edit/main/docs-src/content/docs/docs/cli/configuration.md)
- [Report an unsupported pattern](https://github.com/flaglint/flaglint/issues/new?template=unsupported_pattern.yml)
- Next: [Report Formats](/docs/cli/report-formats/)
