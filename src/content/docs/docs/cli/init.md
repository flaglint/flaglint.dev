---
title: flaglint init
description: Scaffold a flaglint.config.json with all fields set to their defaults.
lastUpdated: 2026-07-01
---

`flaglint init` writes a `flaglint.config.json` to the current directory with every configuration field set to its default value. Run it once to get a starting config you can edit — no manual JSON authoring required.

## Command

```bash
npx flaglint init
```

## Options

| Option | Description |
| --- | --- |
| `--output <path>` | Write the config to a different filename or path (default: `flaglint.config.json`). |
| `--force` | Overwrite an existing config file without error. |

## Config Template Written

```json
{
  "include": ["**/*.{ts,tsx,js,jsx}"],
  "exclude": [
    "**/node_modules/**",
    "**/dist/**",
    "**/build/**",
    "**/.next/**",
    "**/coverage/**",
    "**/*.d.ts"
  ],
  "provider": "launchdarkly",
  "minFileCount": 0,
  "wrappers": [],
  "openFeatureClientBindings": [],
  "outputDir": "."
}
```

After writing, `flaglint init` prints a short explanation of each field to stderr so you know what to change.

## Config Search Order

FlagLint loads the first matching file from this list:

```text
.flaglintrc
.flaglintrc.json
flaglint.config.json
```

If you write to `flaglint.config.json` (the default) and `.flaglintrc` already exists in the same directory, FlagLint will load `.flaglintrc` instead — it has higher precedence. `flaglint init` warns you if this situation would occur.

Use `--output .flaglintrc` to write to the highest-precedence path instead, or `--config <path>` at runtime to use any file regardless of name.

## Exit Codes

| Code | Meaning |
| --- | --- |
| `0` | Config file written successfully. |
| `2` | Config file already exists and `--force` was not passed. |

## Examples

```bash
# Write flaglint.config.json in the current directory
npx flaglint init

# Write to .flaglintrc (highest-precedence name)
npx flaglint init --output .flaglintrc

# Overwrite an existing config without prompting
npx flaglint init --force

# Write to a specific path
npx flaglint init --output config/flaglint.json
```

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/cli/init.md)
- [Report an unsupported pattern](https://github.com/flaglint/flaglint/issues/new?template=unsupported_pattern.yml)
- Next: [Configuration](/docs/cli/configuration/)
