---
title: "Flipt's LaunchDarkly Migration Guide Recommends FlagLint"
description: "Flipt's official guide for migrating from LaunchDarkly SDK to OpenFeature recommends FlagLint for codebase analysis. Here's how the two tools fit together."
date: 2026-07-05
authors:
  - name: Krishan Sharma
    title: Founder and maintainer of FlagLint
    url: https://www.linkedin.com/in/krishansha/
tags: ["flipt", "launchdarkly", "openfeature", "migration", "open-source"]
---

[Flipt](https://flipt.io) is an open-source feature flag platform — one of the most popular self-hosted alternatives to LaunchDarkly. Their official documentation includes a guide for migrating a Node.js application from the LaunchDarkly SDK to OpenFeature, and that guide now [recommends FlagLint](https://docs.flipt.io/v1/guides/migration/launchdarkly/openfeature#optional-use-flaglint) for the codebase analysis step.

Here's what that looks like in practice and why it makes sense.

<!-- Image 1 (intro): "shallow focus photography of computer codes" by Shahadat Rahman
     Source: https://unsplash.com/photos/shallow-focus-photography-of-computer-codes-BfrQnKBulYQ
     Alt: Close-up of TypeScript/JavaScript code on a monitor, representing a codebase undergoing migration -->

## The migration Flipt's guide covers

Flipt supports OpenFeature through its official provider — so a team moving away from LaunchDarkly can route flag evaluation through the OpenFeature API to Flipt's provider instead. The flag evaluation backend changes, but the application code uses the same vendor-neutral API regardless of which provider is behind it.

The challenge isn't the provider setup. It's the application code. A typical Node.js service has dozens of direct LaunchDarkly SDK calls — `boolVariation`, `stringVariation`, `jsonVariation` — all using LaunchDarkly's argument order. OpenFeature's equivalent methods take arguments in a different order, and a naive find-and-replace will silently swap your fallback values and context in every call site.

That's the problem FlagLint solves before you write a single line of migration code.

## How FlagLint fits into the Flipt workflow

Flipt's guide recommends running three FlagLint commands as an optional but practical first step for larger codebases:

```bash
# See every direct LaunchDarkly call site in your codebase
npx flaglint scan ./src

# Get a migration readiness score and per-flag risk classification
npx flaglint audit ./src

# Preview the exact rewrites FlagLint will make, without touching any files
npx flaglint migrate ./src --dry-run
```

The dry-run output shows you exactly which call sites will be rewritten and what the rewritten code looks like — before anything changes. Call sites that FlagLint cannot safely rewrite (dynamic flag keys, detail methods, bulk evaluation) are reported for manual review and left untouched.

After reviewing the diff, `--apply` writes the changes. Then you wire in the Flipt OpenFeature provider, run your tests, and enforce the boundary in CI with `flaglint validate`.

<!-- Image 2 (middle): "a person typing on a laptop computer" by Kaitlyn Baker
     Source: https://unsplash.com/photos/a-person-typing-on-a-laptop-computer-vZJdYl5JVXY
     Alt: A developer running CLI commands in a terminal window -->

## Why this pairing makes sense

FlagLint and Flipt don't overlap. FlagLint analyzes and rewrites your application's call sites. Flipt evaluates your flags at runtime through an OpenFeature provider. They're doing completely different jobs at different layers of the stack.

What they share is a user: someone who has decided to move away from vendor lock-in and is doing the actual migration work. FlagLint handles the static analysis half; Flipt handles the runtime half.

If you're in that position — evaluating Flipt as a LaunchDarkly replacement and trying to understand the migration scope — `flaglint audit` is a good first step. It gives you a concrete inventory and readiness score before you commit to any approach.

```bash
npx flaglint@latest audit ./src
```

No API key, no source upload, runs locally in under a minute. See the [full migration guide](/docs/guides/launchdarkly-to-openfeature-nodejs/) for everything that comes after the audit.

<!-- Image 3 (end): "black laptop computer turned on" by Markus Spiske
     Source: https://unsplash.com/photos/black-laptop-computer-turned-on-C0koz3G1I4I
     Alt: A laptop displaying a clean terminal, representing a completed migration workflow -->
