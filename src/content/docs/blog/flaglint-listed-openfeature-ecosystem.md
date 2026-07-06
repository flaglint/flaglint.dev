---
title: "FlagLint Is Now Listed on the OpenFeature Ecosystem"
description: "FlagLint is now listed in the OpenFeature ecosystem directory as a JavaScript/TypeScript integration. Here's what that means for teams evaluating OpenFeature tooling."
date: 2026-07-05
authors:
  - name: Krishan Sharma
    title: Founder and maintainer of FlagLint
    url: https://www.linkedin.com/in/krishansha/
tags: ["openfeature", "launchdarkly", "ecosystem", "cncf", "migration"]
---

FlagLint is now listed in the [OpenFeature ecosystem directory](https://openfeature.dev/ecosystem/?instant_search%5BrefinementList%5D%5Btype%5D%5B0%5D=Integration) as one of two integrations in the JavaScript/Server category.

I want to be straightforward about what that means — and what it doesn't.

## What the listing is

The OpenFeature ecosystem is a directory maintained by the OpenFeature project (a CNCF incubating standard) where providers, SDKs, hooks, and integrations can be listed. It is not an award or a certification. It is a discovery page. Teams that are already evaluating OpenFeature — researching providers, looking for tooling, trying to understand the ecosystem — land there.

Being listed means that those teams will now find FlagLint when they filter for integrations. That matters because the people who need FlagLint most are exactly the people who are actively thinking about OpenFeature.

## Why I submitted

The OpenFeature standard is the reason FlagLint exists. The whole problem FlagLint solves — the argument-order inversion between LaunchDarkly's `boolVariation(key, ctx, default)` and OpenFeature's `getBooleanValue(key, default, ctx)` — only surfaces when you are trying to move to OpenFeature. If teams weren't adopting OpenFeature, there would be no migration to get wrong.

So it made sense to be in the directory where those teams are looking. Not to market FlagLint as a product, but to be findable at the point in the journey where someone is asking "what tooling exists around OpenFeature for LaunchDarkly migration?"

## What FlagLint does in the context of OpenFeature

FlagLint is not an OpenFeature SDK or provider. It doesn't evaluate flags. What it does is sit at the boundary between your existing LaunchDarkly codebase and the OpenFeature world you're moving toward.

Specifically:

- **Audit** — inventory every direct LaunchDarkly SDK call, classify each one by migration risk, produce a readiness score
- **Migrate** — preview and apply proven-safe call-site rewrites that transpose arguments correctly and rename methods atomically
- **Validate** — enforce in CI that no new direct LaunchDarkly calls land once you've drawn the boundary

None of that requires a network connection, an API key, or access to your LaunchDarkly environment. It's all static analysis on your source code, running locally.

## What's next

The listing covers the JavaScript/TypeScript CLI. A Go implementation is on the roadmap for teams running Go services alongside Node.js — same analysis, same safety model, different language target.

If you're in the process of moving to OpenFeature and want to understand your current exposure before touching any code, the [audit command](/docs/cli/audit/) is the right starting point.

```bash
npx flaglint@latest audit ./src
```

It runs in under a minute on most codebases and doesn't touch any files.
