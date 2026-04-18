---
section: "Interfaces"
number: 12
category: actionable
rules:
  - interfaces (intro)
  - avoid-unnecessary-interfaces
  - interface-ownership-and-visibility
  - designing-effective-interfaces
sources:
  - https://google.github.io/styleguide/go/best-practices#interfaces
---

# Section 12 — Interfaces (Best Practices)

## What this section covers

Three deep-dive recommendations on interface design — when *not* to create one, who should own and export it, and how to design the ones you do create. Best Practices builds on the brief `decisions-review:05-language.md#interfaces` rule with detailed treatment of consumer-defined interfaces, the "accept interfaces, return concrete types" idiom (with explicit exceptions), and the rare-but-real case for breaking circular dependencies via interfaces.

> [!IMPORTANT]
> The single most-quoted rule in this section: **"the consumer defines the interface."** Interfaces should generally live in the package that *uses* them, not the package that *implements* them. This is the inverse of what many developers from Java/C# backgrounds expect.

## Overview

> Interfaces in Go are powerful but can be overused or misunderstood. Because Go interfaces are satisfied implicitly, they are a structural tool rather than a declarative one. The following guidance provides the best practices for how to design and return interfaces in Go without over-engineering your codebase.

> Refer to [Decisions' section on interfaces](https://google.github.io/styleguide/go/decisions#interfaces) for a summary.

(Sibling: `decisions-review:05-language.md#interfaces`.)

## #avoid-unnecessary-interfaces — Avoid unnecessary interfaces

> The most common mistake is creating an interface before a real need exists.

### Three "don't"s

> 1. **Don't confuse the concept with the keyword:** Just because you are designing a "service" or a "repository" or similar pattern doesn't mean you need a named interface type (e.g., `type Service interface`). Focus on the behavior and its concrete implementation first.

> 2. **Reuse existing interfaces:** If an interface already exists, especially in generated code, like a RPC client or server, use it. Do not wrap a generated RPC code in a new, manual interface just for the sake of abstraction or testing. Use real transports instead. (See `best-practices-review:09-tests.md#use-real-transports`.)

> 3. **Don't define back doors only for tests:** Do not export a test double implementation of an interface from an API that consumes it. Instead, prefer to design the API so that it can be tested using the public API of the real implementation.
>
> Every exported type increases the cognitive load for the reader. When you export a test double alongside the real implementation, you force the reader to understand three entities (the interface, the real implementation, and the test double) instead of one.
>
> Export an interface for a test double when you have a material need to support substitution.

### When it does make sense to create an interface

> 1. **Multiple implementations:** When there are two or more concrete types that must be handled by the same logic (e.g., something that operates with both `json.Encoder` and `gob.GobEncoder`), the API consumer could define an interface.

> 2. **Decoupling packages:** To break circular dependencies between two packages, an API producer could define an interface.
>
> **Caution:** Carefully observe guidance on Package Size. Introducing interfaces to break dependency cycles is often a signal of improperly structured packages.

> 3. **Hiding complexity:** When a concrete type has a massive API surface, but a specific function only needs one or two methods, an API consumer may define an interface.

### Decision tree

| Situation | Create an interface? |
|---|---|
| Single concrete implementation, no test substitution need | **No** — concrete type only |
| Two+ concrete types that callers handle uniformly | **Yes** — consumer-side |
| RPC client/server with a generated interface available | **Use the generated one**; don't wrap |
| Need a test double for a single-implementation API | **No** — design for testing the real implementation; only add the interface if you have a *material* need |
| Function only needs 2 methods of a type with 30 methods | **Yes** — consumer-side, narrow interface |
| Two packages would need to import each other | **Yes** — interface to break the cycle (but reconsider the package boundaries first) |

### Smells

- `type FooService interface { ... }` defined in the package that implements it, with one implementation and no real substitution use case.
- `type MockFoo` exported from the production package solely for tests — should not be visible outside tests; usually an unnecessary interface.
- Wrapping a generated gRPC client (`pb.FooClient`) in a hand-written `FooClient` interface — use the generated one with `bufconn` test backend.
- `type XRepository`, `type XManager`, `type XService` interfaces defined preemptively before there's a second implementation.
- Single-implementation interface with wide method set "in case we want to swap later" — YAGNI; add when the second implementation actually appears.

---

## #interface-ownership-and-visibility — Interface ownership and visibility

> 1. **Do not export interface types unnecessarily:** If an interface is only used internally within a package to satisfy a specific logic flow, keep the interface unexported. Exporting an interface commits you to maintaining that API for external callers.

> 2. **The consumer defines the interface:** In Go, interfaces generally belong in the package that uses them, not the package that implements them. The consumer should define only the methods they actually use, adhering to the idea that *the bigger the interface, the weaker the abstraction*.

So the default ownership rule: **interface in the consumer package, implementation in the producer package**. Implicit satisfaction makes this work — the producer doesn't even need to know the consumer's interface exists.

### When the producer SHOULD export the interface

> There are common scenarios where it often makes sense for the producer (the package providing the logic) to export the interface:

#### a. The interface IS the product

> When a package's primary purpose is to provide a common protocol that many different implementations must follow, the producer defines the interface. For example, `io.Writer`, `hash.Hash`. The concept of "protocol" includes aspects like documentation about critical behaviors (e.g., expected use case, edge cases, concurrency) that need to be centrally and canonically explicated.
>
> Another prominent example of this is generated interfaces from protobuf. It doesn't abstract a specific behavior, it defines a boundary. Its purpose is to ensure that your server implementation exactly matches the schema defined in the `.proto` file. Here, the interface serves as a rigid legal contract between the service and its clients.

> For large systems, if the interface lives inside a huge implementation package, every client is forced to import the entire world just to reference the interface. You may define the interface in a standalone, implementation-free package, avoiding unnecessary symbols and potential circular dependencies. This is also the same philosophy used by generated code from protobuf.

#### b. Prevent interface bloat

> In large codebases, maintenance becomes difficult if numerous packages utilize the same `AuthService` while each defining an identical `type Authorizer interface`. While Go often favors a little copying over a little dependency, keep in mind that maintaining perfectly mirrored interfaces (see point above) across many packages can create an unnecessary burden.

#### c. Resolve circular dependency

(See the example in `#designing-effective-interfaces` below.)

### Smells

- Exported interface in a package whose only use is by code inside that same package.
- Producer-side interface (`type Foo interface { ... }` in the implementing package) when the consumer would normally define a smaller, focused interface.
- 30 packages each defining their own `type Authorizer interface` with the same shape — consolidate into a single producer-defined interface in a thin "contract" package.
- Test-only interface exported from production code — usually not needed; test the real implementation.
- Producer interface defined in the same package as a heavy implementation, forcing consumers to import the heavy package just to name the interface — extract into a standalone implementation-free contract package.

---

## #designing-effective-interfaces — Designing effective interfaces

### 1. Keep interfaces small

> The larger the interface, the harder it is to implement and to write code that takes advantage of it. Small interfaces are easier to compose into larger ones if needed.

(This is the literal Go proverb.) Compose small interfaces (`io.ReadCloser = io.Reader + io.Closer`) rather than declaring monolithic ones.

### 2. Documentation

> Treat every interface as the "user manual" for your abstraction. The depth of your documentation should be proportional to the interface's cognitive load, not just the count of its methods.

| Interface kind | Documentation strategy |
|---|---|
| **Single-method interfaces** (`io.Writer`) | Document the type itself (contract, edge cases, expected errors) |
| **Multi-method interfaces** | Each individual method requires its own documentation |
| **Unexported interfaces** | Consider documenting them anyway — they're often the glue holding internal logic together; uncommented they become mystery code |

### 3. Accept interfaces, return concrete types

> Returning a concrete type allows the caller to use the full functionality of the value without being locked into a specific interface abstraction.

This is the canonical Go idiom. `decisions-review:05-language.md#interfaces` states it more briefly; here Best Practices spends time on the *exceptions*.

### Three exceptions where returning an interface IS idiomatic

#### a. Encapsulation

> While interfaces cannot strictly hide exported methods (as they remain accessible via type assertions), returning an interface is a powerful tool for limiting the default API surface and guiding the caller's behavior. The most common example is the `error` interface; you almost never return a concrete error type like `*MyCustomError`.

Example — a `ThrottledReader` with a dangerous-to-call-externally `Refill` method:

```go
// Good:
type ThrottledReader struct {
    source     io.Reader
    limit      int  // bytes per second
    balance    int  // current allowance of bytes
    lastRefill time.Time
}

// Read implements the io.Reader interface with rate-limiting logic.
func (t *ThrottledReader) Read(p []byte) (int, error) { ... }

// Refill manually adds tokens to the bucket.
// INTERNAL USE ONLY: Calling this from outside breaks the rate limit logic.
func (t *ThrottledReader) Refill(amount int) {
    t.balance = min(t.balance + amount, t.limit)
}

// New returns the io.Reader with rate-limiting.
func New(r io.Reader, bytesPerSec int) io.Reader {
    return &ThrottledReader{
        source:     r,
        limit:      bytesPerSec,
        balance:    bytesPerSec,
        lastRefill: time.Now(),
    }
}
```

The constructor returns `io.Reader`, not `*ThrottledReader`, so the typical caller doesn't see `Refill`. An internal `AggregateReader` coordinator that legitimately needs to call `Refill` can type-assert; everyone else gets the safe surface.

> **Caution:** Before returning an interface to hide implementation, ask: "Would a user calling these extra methods actually break the system's integrity or meaningfully limit maintainability?" If the extra details allow the user to bypass safety checks, or if exposing the concrete type makes it impossible to change the underlying provider later without a breaking change, you may return an interface. **Do not rotely encapsulate without reason.**

#### b. Certain patterns (factory, strategy, command, chaining)

> If a function is designed to return one of several different concrete types based on decisions made at runtime, it must return an interface.

```go
// Good:
func NewWriter(format string) io.Writer {
    switch format {
    case "json":
        return &jsonWriter{}
    case "xml":
        return &xmlWriter{}
    default:
        return &textWriter{}
    }
}
```

Chaining APIs are another natural fit:

```go
// Good:
type Client interface {
    WithAuth(token string) Client
    Do(req *Request) error
}
```

Allows `client.Do(req)` or `client.WithAuth("token").Do(req)` polymorphically.

> These patterns are guidelines, not rules. Avoid forcing an interface if a single, robust concrete type can handle the abstraction internally. For example, the standard `database/sql` library exports a single concrete `DB` type instead of forcing an interface to handle types like `MySQLDB` and `OracleDB`.

#### c. Avoiding circular dependencies

> If returning a concrete type would require importing a package that already imports your current package, you must return an interface to break the circular dependency.

```go
// Bad:
package app

import "myproject/plugin"

type Config struct {
    APIKey string
}

func Start() {
    p := plugin.New()
}
```

```go
// Bad:
package plugin

import "myproject/app"  // ERROR: Import cycle!

func New() *app.Config {
    return &app.Config{APIKey: "secret"}
}
```

`plugin.New` can't return `*app.Config` because that would import `app`, which imports `plugin`. Solution: have `plugin.New` return a small interface defined in `plugin`:

```go
// Good:
package plugin

type Configurer interface {
    APIKey() string
}

type localConfig struct {
    key string
}

func (c localConfig) APIKey() string { return c.key }

func New() Configurer {
    return &localConfig{key: "secret"}
}
```

```go
package app

import "myproject/plugin"

func Start() {
    conf := plugin.New()  // 'conf' is now a Configurer interface
    fmt.Println(conf.APIKey())
}
```

> **Caution:** Carefully observe guidance on Package Size. Introducing interfaces to break dependency cycles is often a signal of improperly structured packages. Consolidated packages are often preferred over too many too small packages that fail to stand on their own.

(See `best-practices-review:02-package-size.md`.)

### Smells

- Function returns an interface where a concrete type would expose a method the caller would benefit from.
- Function returns `interface{}` (or `any`) where a typed return would do.
- Interface declared with 8+ methods — split or compose.
- Multi-method interface where the methods have no doc comments.
- Test for a function checking concrete-type behavior via type assertion (`v.(*RealType).InternalMethod()`) — sign that the return type was over-encapsulated.
- "Factory" function returning an interface but with one branch — collapse to the concrete type.
- Library breaking a circular import via an interface that "shouldn't have been needed" — investigate whether package boundaries are wrong.

---

## Consolidated review checklist

### Avoid unnecessary interfaces

- [ ] No interface declared without a real consumer / multiple-implementation / hiding-complexity reason.
- [ ] Generated RPC interfaces reused, not wrapped in hand-written ones.
- [ ] No exported test-double interfaces from production packages without a material substitution need.

### Ownership & visibility

- [ ] Interfaces live with their **consumers** by default, not their producers.
- [ ] Interfaces only kept exported when a real external contract requires it.
- [ ] Producer-defined interfaces reserved for: protocol/contract packages (`io.Writer`-shaped), preventing widespread interface duplication, breaking circular deps.
- [ ] Large producer interfaces extracted into standalone contract packages so consumers don't import implementation weight.

### Designing effective interfaces

- [ ] Interfaces small (1-3 methods typical); large ones composed from smaller pieces.
- [ ] Documentation depth matches cognitive load (single-method = type doc; multi-method = each method).
- [ ] Unexported interfaces still documented when they hold non-trivial logic.
- [ ] **Accept interfaces, return concrete types** by default.
- [ ] Interface return type used only for the named exceptions: encapsulating dangerous internals, runtime-polymorphic factory/strategy patterns, breaking circular deps.
- [ ] Encapsulation-via-interface reviewed: does the hidden method actually break correctness if exposed? If not, return concrete.

## Suggested finding phrasing (advisory)

- "`type FooService interface` defined in `pkg/foo` with one implementation and no consumer asking for it — Best Practices §avoid-unnecessary-interfaces: drop the interface; consumers can define one if/when they need it."
- "`MockFoo` interface exported from production package — Best Practices §avoid-unnecessary-interfaces: don't export test-only abstractions; design tests to use the real implementation."
- "Hand-rolled `type FooClient interface` wraps the generated `pb.FooClient` — Best Practices: use the generated interface (with `bufconn` for test transport — see `best-practices-review:09-tests.md#use-real-transports`)."
- "Interface `Authorizer` defined in `pkg/auth` (the implementer) — Best Practices §interface-ownership-and-visibility: consumer should define it. Move to the consumer package; let `auth.Service` satisfy it implicitly."
- "10 different consumer packages each define their own `type Authorizer interface` with the same 3 methods — Best Practices: producer-side definition justified to prevent the duplication burden."
- "Interface `Storage` has 12 methods — Best Practices §designing-effective-interfaces: compose smaller interfaces (`Reader`, `Writer`, `Closer`) instead."
- "Function `func New() Logger` (interface) but only ever returns `*FileLogger` — Best Practices: return the concrete `*FileLogger` so callers can use `*FileLogger`-specific methods if they need to."
- "Function returns interface to hide a `Refresh()` method that's safe to call externally — Best Practices: \"Do not rotely encapsulate without reason\" — return concrete unless calling the method actually breaks correctness."
- "Interface introduced solely to break a circular import between `pkg/a` and `pkg/b` — Best Practices: this is allowed, but per §package-size, consider whether the two packages should merge."

## Sources

- <https://google.github.io/styleguide/go/best-practices#interfaces> — Google Go Best Practices, Interfaces section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#interfaces> — Decisions §interfaces (sibling skill `decisions-review:05-language.md#interfaces`)
- <https://google.github.io/styleguide/go/guide#simplicity> — Style Guide §simplicity (referenced for "avoid premature abstraction")
- <https://google.github.io/styleguide/go/guide#least-mechanism> — Style Guide §simplicity / least mechanism
- <https://go-proverbs.github.io/> — Rob Pike's Go Proverbs (*the bigger the interface, the weaker the abstraction*; *a little copying is better than a little dependency*)
- <https://pkg.go.dev/io#Writer> — `io.Writer` (canonical small-interface example)
- <https://pkg.go.dev/database/sql#DB> — `database/sql.DB` (cited as "single concrete type, no interface" example)
