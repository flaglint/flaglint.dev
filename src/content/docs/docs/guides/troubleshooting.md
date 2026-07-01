---
title: Troubleshooting
description: Common FlagLint setup and migration issues — 0 flags detected, config not found, React SDK, apply skipping files, and CI failures.
lastUpdated: 2026-07-01
---

## No LaunchDarkly SDK usage detected

FlagLint scans but reports 0 flag usages.

**Most common causes:**

**Wrong directory.** FlagLint scans the directory you pass (defaulting to `.`). If your SDK usage
is under `./src`, run:
```bash
npx flaglint@latest scan ./src
```

**SDK import not recognized.** FlagLint only detects Node.js server SDK imports:
- `launchdarkly-node-server-sdk`
- `@launchdarkly/node-server-sdk`

To verify files are being scanned at all, check `scannedFiles` in JSON output:
```bash
npx flaglint@latest scan ./src --format json | jq '.scannedFiles'
```
If `0`, your `include` patterns aren't matching. Check your `flaglint.config.json`.

**Using a custom wrapper.** If your team wraps the LD SDK in a shared utility, FlagLint won't
detect the wrapper calls unless you declare them. Add to your config:
```json
{ "wrappers": ["myGetFlag", "evaluateFeature"] }
```

---

## Config file not found

FlagLint runs with defaults even though you have a config file.

FlagLint searches for `.flaglintrc`, `.flaglintrc.json`, and `flaglint.config.json` in the
**current working directory** (first match wins). If you run from a different directory than
where your config file lives, it won't be found.

**Fix:** Use `-c` to pin the config path explicitly:
```bash
npx flaglint@latest scan ./src -c ./path/to/flaglint.config.json
```
Or always run FlagLint from the project root where your config file lives.

---

## React SDK patterns not auto-migrated

FlagLint detects React SDK hooks (`useFlags`, `useLDClient`, `useVariation`) in `scan` output,
but `flaglint migrate --apply` doesn't transform them.

This is intentional. React SDK → OpenFeature React SDK migration requires changes to component
trees, context providers, and hook call sites that involve UI judgment calls FlagLint cannot
safely automate. Auto-migration is scoped to the Node.js server SDK only.

React SDK usages appear in the inventory and migration plan as **manual review items**. Use
the plan as a checklist and migrate those call sites by hand.

---

## `--apply` shows 0 transformed files

`flaglint migrate --apply` runs but reports "0 call-sites transformed" or "0 file(s) affected."

The migrator only transforms calls where it can identify the OpenFeature client to substitute.
If `openFeatureClientBindings` is not configured, it cannot generate the replacement call and
skips those files.

**Fix:** Add your OpenFeature client variable name to `flaglint.config.json`:
```json
{
  "openFeatureClientBindings": ["openFeatureClient"]
}
```

Run `flaglint migrate --dry-run` first to preview what would be transformed.

---

## `--apply` Skips Files

`migrate --apply` requires a proven OpenFeature client binding per file. Add a local
`OpenFeature.getClient()` binding or configure imported shared clients with
`openFeatureClientBindings` in your config. Files without a resolvable binding are listed in
the "skipped" output with the reason.

---

## Dirty Working Tree

`migrate --apply` refuses to edit a dirty git working tree by default. Commit or stash your
changes first, or pass `--allow-dirty` only when you intentionally accept the risk of applying
on uncommitted work.

---

## validate passes locally but fails in CI

`flaglint validate` exits 0 locally but exits 1 in CI with the same code.

**Likely cause:** Different working directories or config files. CI may run from the repo root
while local runs from a subdirectory, picking up different configs or different `include` patterns
that match different files.

**Fix:** Pin the directory and config explicitly in both environments:
```bash
npx flaglint@latest validate ./src --no-direct-launchdarkly -c ./flaglint.config.json
```

---

## CI Validation Fails After Partial Migration

Use `validate --no-direct-launchdarkly` only after the relevant source path has migrated.
Provider and bootstrap files that legitimately call the LD SDK directly can be excluded:
```bash
npx flaglint@latest validate ./src --no-direct-launchdarkly \
  --bootstrap-exclude "src/provider/**" \
  --bootstrap-exclude "src/bootstrap/ld-init.ts"
```

---

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/guides/troubleshooting.md)
- [Report an unsupported pattern](https://github.com/flaglint/flaglint/issues/new?template=unsupported_pattern.yml)
- Next: [OpenFeature Provider](/docs/integrations/openfeature-provider/)
