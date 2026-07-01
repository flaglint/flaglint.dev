---
title: How FlagLint Works
description: The import-verified static analysis model behind FlagLint's inventory, migration previews, and validation — and why it produces zero false positives on flag detection.
lastUpdated: 2026-07-01
---

FlagLint performs local static analysis. It parses JavaScript and TypeScript source files,
tracks LaunchDarkly client provenance from supported Node.js server SDK imports, and records
evaluation call sites. It never executes application code and never calls LaunchDarkly APIs.

## The import-verified approach

Most flag scanners work by searching for function names like `variation`, `getFlag`, or `ldClient`
across your codebase. This produces false positives whenever an unrelated library happens to use
the same names — breaking CI and eroding trust in the tool.

FlagLint uses **import-verified** detection. A call site is only flagged as a LaunchDarkly
evaluation if the tool can trace a complete provenance chain:

1. A known LD SDK package is imported (`launchdarkly-node-server-sdk` or `@launchdarkly/node-server-sdk`)
2. The SDK's `init()` return value is bound to a variable
3. A `.variation()`, `.boolVariation()`, or equivalent method is called on that specific variable

No import proof → no flag. A function coincidentally named `variation()` from an animation library
is never flagged.

### Concrete example

```typescript
// ❌ grep for "variation" would match this — FlagLint does NOT
import { variation } from 'some-animation-library'
const opacity = variation('fade', 0, 1)

// ✅ FlagLint correctly identifies only this:
import { init } from 'launchdarkly-node-server-sdk'
const client = init(sdkKey, config)
const value = client.variation('my-flag', ctx, false) // ← flagged
```

The variable name does not matter. If `client` is renamed to `ldClient`, `ld`, or `featureFlags`,
FlagLint tracks the binding and still identifies the call. Conversely, a variable named `ldClient`
that was never assigned an LD `init()` return value is ignored.

### Why this matters for migration

Name-heuristic scanners produce false positives that fail CI for code that has nothing to do
with feature flags. Teams either disable the rule or accept noise — both outcomes break the
migration workflow. Import-verified scanning means the inventory is trustworthy: every finding
is a real LaunchDarkly SDK call, and the count is accurate enough to drive engineering planning.

## Analysis Pipeline

<pre class="mermaid">
flowchart LR
    A["Source files"] --> B["AST parse"]
    B --> C["LD client\nprovenance"]
    C --> D["Evaluation\ninventory"]
    D --> E["Migration\ninventory"]
    E --> F["Report /\ndiff / SARIF"]
</pre>

### Phase 1 — Collect

FlagLint uses a TypeScript AST parser (`@typescript-eslint/typescript-estree`) to parse every
file matched by your `include`/`exclude` config. For each file it:

- Resolves all import and require statements
- Identifies LD SDK imports and traces `init()` call bindings
- Identifies custom wrapper functions listed in your `wrappers` config
- Records every evaluation call site with its file, line, flag key, and call type

### Phase 2 — Analyze

Against the collected inventory, FlagLint:

- Detects staleness signals (flags referenced in only one place, single-variant patterns, commented-out removal notes)
- Scores migration readiness (safely automatable vs. requires manual review)
- Identifies dynamic flag keys (runtime-computed strings that can't be statically resolved)

### Phase 3 — Report

FlagLint generates stable fingerprints for each finding
(`launchdarkly:callType:flagKey:normalizedPath`) and produces:

- Structured inventory reports (`scan` — JSON, Markdown, HTML, SARIF)
- Reviewable migration diffs (`migrate --dry-run`)
- Guarded source edits (`migrate --apply`)
- Policy enforcement reports (`validate` — text, SARIF)
- Baseline files for tracking new findings over time

## What FlagLint does not do

- Execute application code or make network calls
- Evaluate flag values at runtime
- Modify files without `migrate --apply` being explicitly passed
- Detect usage of browser SDKs (`launchdarkly-js-client-sdk`) — only the Node.js server SDKs

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/concepts/how-flaglint-works.md)
- [Report an unsupported pattern](https://github.com/flaglint/flaglint/issues/new?template=unsupported_pattern.yml)
- Next: [Safety Model](/docs/concepts/safety-model/)
