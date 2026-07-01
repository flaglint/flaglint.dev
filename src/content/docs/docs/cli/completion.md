---
title: flaglint completion
description: Output a shell completion script for bash, zsh, or fish.
lastUpdated: 2026-07-01
---

`flaglint completion` prints a shell completion script to stdout. Source it once and your shell will tab-complete subcommands, flags, and flag values (format choices, shell names, and more).

## Command

```bash
flaglint completion <shell>
```

Supported shells: `bash`, `zsh`, `fish`.

## Installation

### Bash

Load for the current session only:

```bash
source <(flaglint completion bash)
```

Load permanently (add to `~/.bashrc` or `~/.bash_profile`):

```bash
flaglint completion bash >> ~/.bash_completion
```

### Zsh

Load for the current session only:

```bash
source <(flaglint completion zsh)
```

Load permanently (requires `~/.zsh/completions` in your `$fpath`):

```bash
mkdir -p ~/.zsh/completions
flaglint completion zsh > ~/.zsh/completions/_flaglint
```

Then ensure `$fpath` includes the directory before `compinit` runs — add to `~/.zshrc`:

```zsh
fpath=(~/.zsh/completions $fpath)
autoload -Uz compinit && compinit
```

### Fish

```bash
flaglint completion fish > ~/.config/fish/completions/flaglint.fish
```

Fish auto-sources files in `~/.config/fish/completions/` on startup — no extra configuration needed.

## What Gets Completed

- **Subcommands** — `audit`, `scan`, `migrate`, `validate`, `init`, `completion`
- **Flags** — all per-command flags and global flags (`--quiet`, `--verbose`)
- **Flag values** — `--format` values (`json`, `markdown`, `html`, `sarif`, `text`), completion shell names (`bash`, `zsh`, `fish`)
- **File arguments** — `--output`, `--config`, `--baseline`, `--write-baseline` complete against the filesystem

## Examples

```bash
# Print the bash completion script
flaglint completion bash

# Activate immediately in the current bash session
source <(flaglint completion bash)

# Write the zsh completion script to the standard completions directory
flaglint completion zsh > ~/.zsh/completions/_flaglint

# Write the fish completion file
flaglint completion fish > ~/.config/fish/completions/flaglint.fish
```

## Exit Codes

| Code | Meaning |
| --- | --- |
| `0` | Script printed successfully. |
| `2` | No shell argument provided, or unsupported shell name. |

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/cli/completion.md)
- [Report an unsupported pattern](https://github.com/flaglint/flaglint/issues/new?template=unsupported_pattern.yml)
