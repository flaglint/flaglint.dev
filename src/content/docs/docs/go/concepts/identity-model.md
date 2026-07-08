---
title: flaglint-go Identity Model
description: How flaglint-go proves a variable is a LaunchDarkly client — never by name, and across the whole scan, not just one file.
lastUpdated: 2026-07-08
---

flaglint-go's core rule, inherited from flaglint-js: a variable is only treated as a LaunchDarkly client when its identity can be **proven** — never by matching a method or variable name in isolation.

## Why not just search for method names?

Most flag scanners look for calls like `.BoolVariation(...)` or `.variation(...)` anywhere in the source. This produces false positives the moment an unrelated type happens to define a same-named method:

```go
// ❌ a name-only scanner would match this
type FakeClient struct{}
func (c *FakeClient) BoolVariation(key string, ctx interface{}, def bool) (bool, error) {
	return def, nil
}

// ✅ flaglint-go correctly ignores it — FakeClient is never traced back
// to a github.com/launchdarkly/go-server-sdk import
```

flaglint-go proves identity through **import-alias tracing plus constructor-call binding**: a variable is only a client if it traces back, through the AST, to `ld.MakeClient(...)`/`ld.MakeCustomClient(...)` for a package actually imported from `github.com/launchdarkly/go-server-sdk` (v6 or v7) — whatever local alias is used, including dot-imports.

## Whole-scan resolution

Real-world Go code rarely constructs and uses the client in the same place. flaglint-go parses every file in a scan up front and resolves several indirection patterns *across* files — not just within one:

**Struct fields and composite literals** — the client stored into a wrapper struct:
```go
type Integration struct {
	ldClient *ld.LDClient
}

func setup() *Integration {
	client, _ := ld.MakeClient("sdk-key", 5*time.Second)
	return &Integration{ldClient: client}
}
```

**Multi-level field chains**, including through generics — a struct's fields can be declared in a different file (even a different file than where they're used):
```go
type FeatureFlag[T comparable] struct {
	integ *Integration
}

func (f *FeatureFlag[T]) Evaluate() bool {
	v, _ := f.integ.ldClient.BoolVariation("my-flag", nil, false)
	return v
}
```

**Cross-package factory/getter functions** — a package-level singleton getter called from a different package, resolved via real `go.mod`-derived import paths (never a name-based guess):
```go
// package flags
func GetLdClient() *ld.LDClient { /* ... */ }

// package main
client := flags.GetLdClient()
client.BoolVariation("my-flag", nil, false)
```

**Parameter-typed bindings** — a client passed in as a plain parameter, with no assignment to trace at all:
```go
func useClient(client *ld.LDClient) {
	client.BoolVariation("my-flag", nil, false) // bound from the declared type alone
}
```

**Struct fields declared `*ld.LDClient`, with no observed construction anywhere in the scanned tree** — the dominant Go dependency-injection pattern: the client is wired into the struct by an external framework or a constructor not included in what's actually scanned. The field's declared type alone is sufficient proof, the same "trust the signature" principle as parameter-typed bindings above, just applied to a struct field instead:
```go
type FlagService struct {
	ld *ld.LDClient
}

func (s *FlagService) DarkMode(userID string) bool {
	v, _ := s.ld.BoolVariation("dark-mode", ldcontext.New(userID), false) // bound from the field's declared type alone
	return v
}
```

**Composite literals directly initializing a package-level `var`**, never inside any function body:
```go
var svcClient, _ = ld.MakeClient("sdk-key", 5*time.Second)
var svc = &Svc{Client: svcClient} // never inside a function — resolved at the package level

func NewUI(userID string) bool {
	v, _ := svc.Client.BoolVariation("new-ui", ldcontext.New(userID), false)
	return v
}
```

## Phase 2 — `--strict-types`

Phase 1's syntactic tracing is structurally unable to prove a small number of patterns — they need real type information, not just syntax. An opt-in `--strict-types` pass loads the target module with real `go/types` information (via `golang.org/x/tools/go/packages`) and uses it to resolve these additionally. This pass is strictly *additive*: it can only add findings Phase 1 missed, never remove or contradict one Phase 1 already made. Phase 1 remains the default — `flaglint-go audit` continues to work unconditionally, with no build required; `--strict-types` requires the scanned module to actually build.

**Interface satisfaction** — a client known only through an interface type, not the concrete `*ld.LDClient` type itself:
```go
type FlagEvaluator interface {
	BoolVariation(key string, ctx ldcontext.Context, defaultVal bool) (bool, error)
}

func useEvaluator(e FlagEvaluator) {
	e.BoolVariation("my-flag", nil, false) // e's declared type is an interface, not *ld.LDClient —
	                                          // requires go/types to confirm *ld.LDClient satisfies it
}
```

**A factory function returning a wrapper type**, not `*ld.LDClient` itself (transitive factory wrapping):
```go
func NewClientWrapper() *ClientWrapper { /* constructs and wraps a real client */ }

wrapper := NewClientWrapper()
wrapper.inner.BoolVariation("my-flag", nil, false)
```

**A method value passed as an argument into a *different* function** (the "forwarding function" pattern) — the harder remainder beyond the same-function method-value case Phase 1 already resolves:
```go
func callDirect(f func(string, ldcontext.Context, bool) (bool, error), key string, def bool) bool {
	v, _ := f(key, nil, def)
	return v
}

func useDirect(client *ld.LDClient) {
	_ = callDirect(client.BoolVariation, "forwarding-flag", false) // method value crosses a function boundary
}
```

## What stays out of scope

Identity resolution is deliberately conservative — flaglint-go prefers a missed detection (false negative) over a false positive. Patterns not resolved by either Phase 1 or `--strict-types` are documented, not silently guessed at. See [Limitations](/docs/go/reference/limitations/) for the current list.

## Dynamic Key Detection

A flag-key argument is `isDynamic: true` whenever it isn't a string literal — an identifier, a `fmt.Sprintf(...)` call, or string concatenation. Dynamic call sites are always classified high risk, regardless of the underlying method's own risk level, since the key "cannot resolve statically" no matter how simple the method otherwise is.

## Feedback

- [Edit this page on GitHub](https://github.com/flaglint/flaglint.dev/edit/main/src/content/docs/docs/go/concepts/identity-model.md)
- [Report an issue](https://github.com/flaglint/flaglint-go/issues/new)
- Next: [Supported Scope](/docs/go/reference/supported-scope/)
