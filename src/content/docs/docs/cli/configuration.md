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

| Field | Type | Default | Purpose |
| --- | --- | --- | --- |
| `include` | `string[]` | `["**/*.{ts,tsx,js,jsx}"]` | Files to scan. |
| `exclude` | `string[]` | common build/test output | Files to skip. |
| `provider` | `"launchdarkly"` | `"launchdarkly"` | Provider scope. Currently only `"launchdarkly"` is supported — other values exit 2. |
| `minFileCount` | `number` | `0` | Optional local source review heuristic. |
| `wrappers` | `(string \| WrapperObject)[]` | `[]` | Wrapper functions to detect as flag evaluations. Accepts string names or object form (see below). |
| `openFeatureClientBindings` | `string[]` | `[]` | Imported shared client allowlist for `migrate --apply`. |
| `reportTitle` | `string` (optional) | unset | Optional report title injected into HTML and Markdown reports. |
| `outputDir` | `string` | `"."` | Directory for generated report files. |

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

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/cli/configuration.md)
- [Report an unsupported pattern](https://github.com/flaglint/flaglint/issues/new?template=unsupported_pattern.yml)
- Next: [Report Formats](/docs/cli/report-formats/)

## OpenFeature Client Bindings

`openFeatureClientBindings` tells `flaglint migrate --apply` which variables in your code hold
an OpenFeature client. When applying transformations, FlagLint substitutes LD SDK evaluation
calls with the corresponding OpenFeature client method calls — but only when it can identify
the client variable to substitute.

Each entry is a **variable name** (not a package name, not a file path) that FlagLint should
treat as an OpenFeature client.

### Example

Your application code:

```typescript
import openFeatureClient from './platform/of-client'

const value = openFeatureClient.getBooleanValue('my-flag', false)
```

Your config:

```json
{
  "openFeatureClientBindings": ["openFeatureClient"]
}
```

With this configured, `migrate --apply` can generate the correct OpenFeature replacement call.
Without it, files that use an imported client are listed in the "skipped" output.

### Multiple bindings

If different parts of your codebase use different variable names for the OpenFeature client,
list all of them:

```json
{
  "openFeatureClientBindings": ["openFeatureClient", "featureClient", "ofClient"]
}
```

### Difference from wrappers

`openFeatureClientBindings` is for the **OpenFeature client** used in migration output — it
tells FlagLint what to generate, not what to detect. `wrappers` is for **existing LD-compatible
wrappers** — it tells FlagLint what to detect as flag evaluations in your current code.
