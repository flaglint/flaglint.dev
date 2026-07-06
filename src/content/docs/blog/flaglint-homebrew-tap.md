---
title: "FlagLint Is Now Installable via Homebrew"
description: "FlagLint v1.1.0 is available via a Homebrew tap. No Node.js required — one command installs the full CLI on macOS or Linux."
date: 2026-07-05
authors:
  - name: Krishan Sharma
    title: Founder and maintainer of FlagLint
    url: https://www.linkedin.com/in/krishansha/
tags: ["homebrew", "cli", "install", "devops", "nodejs"]
---

Starting with v1.1.0, FlagLint is installable via Homebrew — no Node.js required.

```bash
brew tap flaglint/tap
brew install flaglint
```

That's it. The full CLI is available immediately, including `audit`, `scan`, `migrate`, and `validate`.

<!-- Image 1 (intro): "person using MacBook Pro" by Glenn Carstens-Peters
     Source: https://unsplash.com/photos/person-using-macbook-pro-npxXWgQ33ZQ
     Alt: A developer at a laptop running CLI commands in a terminal -->

## Why this matters

Until now, the only install paths were `npm install -g flaglint` or `npx flaglint@latest`. Both work fine — but both require Node.js 20 or newer. That's a reasonable assumption for application developers, but it's a friction point in a few common situations.

**DevOps and platform engineers** often run tooling audits across repos without a Node.js environment set up. Installing Node just to run one CLI is the kind of thing that gets a tool skipped in favour of whatever's already available.

**Docker-based CI** is the bigger one. A lot of teams run their CI in minimal images — no Node, no npm. Adding a Node install step purely to run `npx flaglint` adds 30-60 seconds to every pipeline run and pulls in a dependency that has nothing to do with the actual application. With Homebrew, a Linux CI job can install FlagLint in a single `brew install` call with no Node dependency.

**Mac developers** who use Homebrew for CLI tools now get `brew upgrade flaglint` like any other tool, without thinking about npm.

<!-- Image 2 (middle): "black screen with code" by Markus Spiske
     Source: https://unsplash.com/photos/black-screen-with-code-cvBBO4PzWPg
     Alt: A dark terminal screen displaying command-line output, representing CLI tool usage -->

## How the tap works

The tap lives at [github.com/flaglint/homebrew-tap](https://github.com/flaglint/homebrew-tap). The formula fetches the published npm tarball directly from the npm registry and wires up the CLI binary.

The formula updates automatically on every release — a GitHub Actions job in the main repo runs after each npm publish, computes the new tarball SHA256, and commits an updated formula to the tap. So `brew upgrade flaglint` will always pull the current release without any manual intervention.

## Getting started

```bash
brew tap flaglint/tap
brew install flaglint
flaglint --version
```

Or if you already have Node.js, `npx flaglint@latest` still works exactly as before. The Homebrew path is an addition, not a replacement.

Once installed, the [quickstart](/docs/quickstart/) walks through the full audit → preview → apply workflow.

<!-- Image 3 (end): "macbook pro on brown wooden table" by Roberto Nickson
     Source: https://unsplash.com/photos/macbook-pro-on-brown-wooden-table-5t1lUr0NmHI
     Alt: A MacBook on a desk, representing macOS CLI tooling with Homebrew -->
