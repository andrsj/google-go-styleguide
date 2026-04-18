---
section: "Global state"
number: 11
category: actionable
rules:
  - global-state (intro)
  - major-forms-of-package-state-apis
  - litmus-tests
  - providing-a-default-instance
sources:
  - https://google.github.io/styleguide/go/best-practices#global-state
---

# Section 11 — Global state (Best Practices)

## What this section covers

How to think about (and mostly avoid) global / package-level state in Go libraries. The doc names three problematic API forms, gives explicit litmus tests for when global state is and isn't safe, and prescribes a constrained pattern for shipping a "default instance" convenience API.

> [!IMPORTANT]
> The Best Practices doc says global state should be approached with **extreme scrutiny**. Findings under this section are typically advisory but high-priority — package-level state is one of the hardest things to refactor away later, so catching it at design review time saves enormous downstream cost.

## Overview

> Libraries should not force their clients to use APIs that rely on global state. They are advised not to expose APIs or export package level variables that control behavior for all clients as parts of their API.

The doc uses "global" and "package level state" synonymously. This section's rules apply to both exported AND unexported package-level state if it shapes client-observable behavior.

> Instead, if your functionality maintains state, allow your clients to create and use instance values.

> **Important:** While this guidance is applicable to all developers, it is most critical for infrastructure providers who offer libraries, integrations, and services to other teams.

### Good — instance-based, dependency-passing

```go
// Package sidecar manages subprocesses that provide features for applications.
package sidecar

type Registry struct{ plugins map[string]*Plugin }

func New() *Registry { return &Registry{plugins: make(map[string]*Plugin)} }

func (r *Registry) Register(name string, p *Plugin) error { /* ... */ }
```

```go
package main

func main() {
    sidecars := sidecar.New()
    if err := sidecars.Register("Cloud Logger", cloudlogger.New()); err != nil {
        log.Exitf("Could not setup cloud logger: %v", err)
    }
    cfg := &myapp.Config{Sidecars: sidecars}
    myapp.Run(context.Background(), cfg)
}
```

The user instantiates a `*sidecar.Registry`, populates it, and passes it down the call chain as an explicit dependency.

> There are different approaches to migrating existing code to support dependency passing. The main one you will use is passing dependencies as parameters to constructors, functions, methods, or struct fields on the call chain.

### Bad — global registry

```go
package sidecar

var registry = make(map[string]*Plugin)

func Register(name string, p *Plugin) error { /* registers plugin in registry */ }
```

This API forces every client into the same shared registry. The doc walks through a concrete failure mode: tests that mutate the global registry leak state into each other.

```go
// Bad — order-dependent tests via shared global state:
func TestEndToEnd(t *testing.T) {
    // SUT relies on the production cloud logger already being registered.
    // ...
}

func TestRegression_NetworkUnavailability(t *testing.T) {
    sidecar.Register("cloudlogger", cloudloggertest.UnavailableLogger)
    // ...
}

func TestRegression_InvalidUser(t *testing.T) {
    // Oops. cloudloggertest.UnavailableLogger is still registered from the previous test.
    // ...
}
```

Go tests run sequentially by default, so the order is `TestEndToEnd` → `TestRegression_NetworkUnavailability` (overrides the global) → `TestRegression_InvalidUser` (now sees the wrong global). This breaks `-run` filtering, parallelism, and sharding.

### The questions global state can't easily answer

The doc enumerates ones a global-state API leaves to the client to figure out:

- What if a client needs **multiple separate instances** in the same process (e.g., multiple servers each with their own plugins)?
- What if a client needs to **replace registered values in tests**?
- What if multiple clients **`Register` under the same name** — who wins? How are errors handled? Is `panic` / `log.Fatal` appropriate at every call site?
- Is there a **specific lifecycle stage** during which `Register` may be called? What if it's called before flags are parsed, in `init`, or after `main`? The lifecycle ambiguity often pushes authors toward `Must`-style abort-on-error, which isn't appropriate for general-purpose libraries (see `best-practices-review:04-error-handling.md#program-initialization`).
- What if the client's **concurrency expectations** don't match the API author's?

> Global state has cascading effects on the health of the Google codebase. Global state should be approached with **extreme scrutiny**.

## #major-forms-of-package-state-apis — Major forms of package state APIs

> Several of the most common problematic API forms are enumerated below.

### 1. Top-level variables (exported or not)

```go
// Bad:
package logger

// Sinks manages the default output sources for this package's logging API.
// This variable should be set at package initialization time and never thereafter.
var Sinks []Sink
```

The "should be set at init time and never thereafter" comment is a red flag — it's a *convention* the API can't enforce.

### 2. Service locator pattern with global locator

> The service locator pattern itself is not problematic, rather **the locator being defined as global**.

The fix isn't to eliminate the pattern; it's to make the locator an instance the client passes around.

### 3. Registries for callbacks and similar behaviors

```go
// Bad:
package health

var unhealthyFuncs []func()

func OnUnhealthy(f func()) {
    unhealthyFuncs = append(unhealthyFuncs, f)
}
```

Same fix: callbacks register to an instance, not a package-level slice.

### 4. Thick-client singletons (backends, storage, data layers)

```go
// Bad:
package useradmin

var client pb.UserAdminServiceClientInterface

func Client() *pb.UserAdminServiceClient {
    if client == nil {
        client = ... // Set up client.
    }
    return client
}
```

> These often pose additional problems with service reliability.

Lazy-initialized singletons are doubly problematic: state + lifecycle ambiguity + concurrency hazard (the `if client == nil` is racy without a mutex).

### Note on legacy code

> Many legacy APIs in the Google codebase do not follow this guidance; in fact, some Go standard libraries allow for configuration via global values. Nevertheless, the legacy API's contravention of this guidance **should not be used as precedent** for continuing the pattern.

> It is better to invest in proper API design today than pay for redesigning later.

So `http.DefaultServeMux`, `log.Default()`, `flag.CommandLine`, etc. exist — but they aren't a license to add new globals.

### Smells

- Exported package-level `var X T` that the API mutates from public functions.
- Lazy-init singleton (`if x == nil { x = ... }`) — unsafe under concurrency, hides lifecycle.
- `package init()` populating package-level state that affects observable behavior.
- Tests that need a `// reset global state` cleanup helper — sign the API has unmanaged global state.
- Library function that takes no instance argument but reads package state internally.
- "Configure once at startup" pattern (`pkg.SetXxx(value)` early, `pkg.GetXxx()` later) — racy and lifecycle-ambiguous.

---

## #litmus-tests — Litmus tests

> [APIs using the patterns above] are unsafe when:
>
> - Multiple functions interact via global state when executed in the same program, despite being otherwise independent (for example, authored by different authors in vastly different directories).
> - Independent test cases interact with each other through global state.
> - Users of the API are tempted to swap or replace global state for testing purposes, particularly to replace any part of the state with a test double, like a stub, fake, spy, or mock.
> - Users have to consider special ordering requirements when interacting with global state: `func init`, whether flags are parsed yet, etc.

If **any** of those apply, the global state is unsafe — refactor to instances.

> Provided the conditions above are avoided, there are a **few limited circumstances under which these APIs are safe**, namely when any of the following is true:
>
> - The global state is logically constant (example: pre-computed lookup tables).
> - The package's observable behavior is stateless. For example, a public function may use a private global variable as a cache, but so long as the caller can't distinguish cache hits from misses, the function is stateless.
> - The global state does not bleed into things that are external to the program, like sidecar processes or files on a shared filesystem.
> - There is no expectation of predictable behavior (example: `math/rand`).

So global state is **safe** mainly when: logically constant, or hidden behind a stateless façade, or scope-bounded to in-process only, or explicitly non-deterministic.

### Sidecar caveat

> Sidecar processes may **not** strictly be process-local. They can and often are shared with more than one application process. Moreover, these sidecars often interact with external distributed systems.

So a "global" that talks to a sidecar isn't safely "in-process" even if the variable is local — the side effects leak.

### Worked example: `image.RegisterFormat`

The doc walks through `image.RegisterFormat` (`image/png`, etc.) as one of the rare *safe* cases:

- Multiple calls to `image.Decode` don't interfere — only `RegisterFormat` mutates state.
- Replacing a registered decoder with a test double is unlikely (real decoders are usually fine in tests).
- Registration collisions are conceivable but rare in practice.
- The decoders themselves are stateless, idempotent, and pure.

So this passes the litmus tests despite using a package-level registry.

### Smells (litmus failures)

- Two seemingly-independent code paths reading the same package-level value — the failures are unsafe per litmus point 1.
- Tests need ordering, isolation, or cleanup because they share package state — point 2.
- API doc explains how to "swap out" the global for testing — point 3.
- API has lifecycle rules ("must be called before X") — point 4.

---

## #providing-a-default-instance — Providing a default instance

> While not recommended, it is acceptable to provide a simplified API that uses package level state if you need to maximize convenience for the user.

If you ship one, follow these constraints:

> 1. The package must offer clients the ability to **create isolated instances** of package types as described above.
> 2. The public APIs that use global state must be a **thin proxy** to the previous API. A good example of this is `http.Handle` internally calling `(*http.ServeMux).Handle` on the package variable `http.DefaultServeMux`.
> 3. This package-level API must only be used by **binary build targets, not libraries**, unless the libraries are undertaking a refactoring to support dependency passing. Infrastructure libraries that can be imported by other packages must not rely on package-level state of the packages they import.
> 4. This package-level API must **document and enforce its invariants** (for example, at which stage in the program's life it can be called, whether it can be used concurrently). Further, it must provide an API to **reset global state to a known-good default** (for example, to facilitate testing).

### Good — proxy + companion `Register` hook

```go
// Good:
package cloudlogger

func New() *Logger { ... }

func Register(r *sidecar.Registry, l *Logger) {
    r.Register("Cloud Logging", l)
}
```

The instance API (`New() *Logger`) is the source of truth. The `Register` helper takes the registry as an explicit parameter — no global involved. A binary's `main` can wire it up, and tests can wire up isolated registries without polluting each other.

### The "default instance" pattern in practice

```go
// Acceptable when constraints above hold:
package mylib

type Client struct{ /* ... */ }

func New() *Client { return &Client{...} }

// DefaultClient is the package-level default; mostly for binary convenience.
var DefaultClient = New()

// Get is a thin proxy to DefaultClient.Get.
func Get(url string) (*Response, error) {
    return DefaultClient.Get(url)
}

// Reset re-creates DefaultClient. Tests use this to isolate.
func Reset() {
    DefaultClient = New()
}
```

The package satisfies the four constraints: instance API exists (`New`); package-level functions are thin proxies (`Get`); intended for binaries; documented invariants + `Reset()` for tests.

### Smells

- "Default instance" pattern with the package-level functions doing real work, not just proxying — refactor real logic onto the type.
- Library imports another library's package-level API instead of taking an instance — violates rule 3.
- No `Reset()` or equivalent provided for tests — violates rule 4.
- The instance type isn't actually exported, so callers can't escape the default — violates rule 1.
- Documentation lacks lifecycle / concurrency invariants for the package-level API.

---

## Decision tree

| Situation | Use |
|---|---|
| New library, designing API now | Instance-based, no globals |
| New library that needs a "convenience" form on top of instance API | Default-instance pattern with all 4 constraints (instance escape, thin proxy, binary-only, documented + reset) |
| Existing library with global state, can refactor | Add instance-based API; deprecate package-level functions; provide migration path |
| Existing library with global state, can't refactor (stable contract) | Document as legacy; don't *extend* the global pattern; don't cite as precedent |
| Stdlib-style "configure once" defaults that are logically constant | Acceptable per litmus tests |

---

## Consolidated review checklist

- [ ] No exported package-level vars used to control client-observable behavior.
- [ ] No registries, callbacks, or thick-client singletons living at package scope.
- [ ] Package types instantiable via a `New()`-style constructor.
- [ ] Tests don't need `// reset global state` cleanups.
- [ ] No lazy-init singleton (`if x == nil { x = ... }`) without proper synchronization.
- [ ] No "configure once at startup" pattern (`pkg.SetXxx`).
- [ ] If a default instance is shipped: instance escape exists, package functions are thin proxies, only used by binaries, documented invariants, `Reset()` for tests.
- [ ] Libraries don't depend on other packages' package-level state.

## Suggested finding phrasing (advisory)

- "`var registry = make(map[string]*Plugin)` at package scope, mutated by `Register(...)` — Best Practices §global-state: refactor to a `Registry` type with a `New()` constructor and methods; pass the instance down the call chain."
- "Lazy singleton `func Client() { if client == nil { client = ... }; return client }` — Best Practices §major-forms: thick-client singleton with race + lifecycle hazards; use an instance returned by `New()`."
- "Tests need to call `pkg.ResetGlobalRegistry()` between cases — Best Practices §litmus-tests (point 2): tests interact through global state; refactor to instances."
- "Library `internal/logger` imports `pkg/logsink` and uses its package-level `DefaultSink` — Best Practices §providing-a-default-instance (rule 3): libraries must not depend on package-level state of their imports; take an instance as a parameter."
- "`init()` populates package-level config from env vars — Best Practices §major-forms: makes lifecycle ambiguous; expose a `New(opts)` constructor instead and wire from `main`."
- "Package ships `DefaultClient` and `Get(url)` proxy but no `Reset()` — Best Practices §providing-a-default-instance (rule 4): tests can't isolate; add a `Reset()` function."

## Sources

- <https://google.github.io/styleguide/go/best-practices#global-state> — Google Go Best Practices, Global state section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#dont-panic> — Decisions §dont-panic (referenced for the lifecycle / panic concern)
- <https://google.github.io/styleguide/go/best-practices#program-initialization> — `best-practices-review:04-error-handling.md#program-initialization` (referenced for `Must`-pattern hazards)
- <https://google.github.io/styleguide/go/guide#local-consistency> — `guide-review:10-local-consistency.md` (referenced — legacy globals don't license new ones)
- <https://pkg.go.dev/image#RegisterFormat> — `image.RegisterFormat` (cited as a *safe* example)
- <https://pkg.go.dev/net/http#Handle> — `http.Handle` (cited as the canonical "thin proxy to default instance" pattern)
