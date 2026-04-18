---
section: "Function argument lists"
number: 7
category: actionable
rules:
  - option-structure
  - variadic-options
sources:
  - https://google.github.io/styleguide/go/best-practices#function-argument-lists
---

# Section 07 — Function argument lists (Best Practices)

## What this section covers

Two patterns for keeping function signatures usable as parameter counts grow: the **option struct** (one struct param carrying all configuration) and **variadic options** (functional options via closures). Both replace the anti-pattern of an ever-growing positional parameter list.

> [!IMPORTANT]
> The general principle is also stated in `decisions-review:05-language.md#func-formatting` (signatures stay on one line; refactor by reducing arguments). This section gives the *two specific refactoring shapes* Best Practices recommends, with criteria for picking between them.

## Overview

> Don't let the signature of a function get too long. As more parameters are added to a function, the role of individual parameters becomes less clear, and adjacent parameters of the same type become easier to confuse.

> When designing an API, consider splitting a highly configurable function whose signature is growing complex into several simpler ones.

So before either of the two refactoring patterns, the first question is: **can this be split into separate functions?** If yes, do that. The two patterns below are for genuinely-configurable single operations.

## #option-structure — Option structure

> An option structure is a struct type that collects some or all of the arguments of a function or method, that is then passed as the last argument to the function or method.

### Benefits

> The struct literal includes both fields and values for each argument, which makes them self-documenting and harder to swap.

> Irrelevant or "default" fields can be omitted.

> Option structs can grow over time without impacting call-sites.

### Bad — long positional signature

```go
func EnableReplication(ctx context.Context, config *replicator.Config,
    primaryRegions, readonlyRegions []string, replicateExisting,
    overwritePolicies bool, replicationInterval time.Duration,
    copyWorkers int, healthWatcher health.Watcher) {
    // ...
}
```

### Good — option struct

```go
type ReplicationOptions struct {
    Config              *replicator.Config
    PrimaryRegions      []string
    ReadonlyRegions     []string
    ReplicateExisting   bool
    OverwritePolicies   bool
    ReplicationInterval time.Duration
    CopyWorkers         int
    HealthWatcher       health.Watcher
}

func EnableReplication(ctx context.Context, opts ReplicationOptions) {
    // ...
}
```

### When to use option structs

> This option is often preferred when some of the following apply:
> - All callers need to specify one or more of the options.
> - A large number of callers need to provide many options.
> - The options are shared between multiple functions that the user will call.

### Important: Contexts are NEVER in option structs

`context.Context` stays as a separate first parameter. Don't put `Ctx context.Context` as a field on the options struct. (See `decisions-review:06-common-libraries.md#contexts`.)

### Smells

- Function signature with 5+ parameters of mixed types — candidate for an option struct.
- Function signature with multiple parameters of the same type adjacent (`(a, b, c, d string)`) — easy to swap; option struct or named-type wrappers eliminate this.
- Option struct with a `Ctx context.Context` field — move `ctx` back to the first positional parameter.
- Option struct with no zero-default-friendly fields — most fields *required* — that's a sign the function is doing too much; consider splitting first.
- Two sibling functions taking nearly-identical option structs — share the struct type instead of duplicating.
- Option struct passed by pointer (`*ReplicationOptions`) when the function doesn't mutate — pass by value.

---

## #variadic-options — Variadic options

> Using variadic options, exported functions are created which return closures that can be passed to the variadic (`...`) parameter of a function. The function takes as its parameters the values of the option (if any), and the returned closure accepts a mutable reference (usually a pointer to a struct type) that will be updated based on the inputs.

### Benefits

> Options take no space at a call-site when no configuration is needed.
>
> Options can accept multiple parameters (e.g. `cartesian.Translate(dx, dy int) TransformOption`).
>
> Packages can allow (or prevent) third-party packages to define their own options.

### Caution

> Using variadic options should only be used when the advantages outweigh the overhead.

The pattern has real cost: an unexported config struct, one exported helper function per option, default merging, doc duplication. Use it when you've genuinely earned it.

### Canonical implementation

```go
type replicationOptions struct {
    readonlyCells       []string
    replicateExisting   bool
    overwritePolicies   bool
    replicationInterval time.Duration
    copyWorkers         int
    healthWatcher       health.Watcher
}

// A ReplicationOption configures EnableReplication.
type ReplicationOption func(*replicationOptions)

// ReadonlyCells adds additional cells that should additionally
// contain read-only replicas of the data.
//
// Passing this option multiple times will add additional
// read-only cells.
//
// Default: none
func ReadonlyCells(cells ...string) ReplicationOption {
    return func(opts *replicationOptions) {
        opts.readonlyCells = append(opts.readonlyCells, cells...)
    }
}

// ReplicateExisting controls whether files that already exist in the
// primary cells will be replicated. Otherwise, only newly-added
// files will be candidates for replication.
//
// Passing this option again will overwrite earlier values.
//
// Default: false
func ReplicateExisting(enabled bool) ReplicationOption {
    return func(opts *replicationOptions) {
        opts.replicateExisting = enabled
    }
}

// ... other options ...

// DefaultReplicationOptions control the default values before
// applying options passed to EnableReplication.
var DefaultReplicationOptions = []ReplicationOption{
    OverwritePolicies(true),
    ReplicationInterval(12 * time.Hour),
    CopyWorkers(10),
}

func EnableReplication(ctx context.Context, config *placer.Config,
    primaryCells []string, opts ...ReplicationOption) {

    var options replicationOptions
    for _, opt := range DefaultReplicationOptions {
        opt(&options)
    }
    for _, opt := range opts {
        opt(&options)
    }
    // ... use options ...
}
```

### Call sites — short for default, expressive for complex

```go
// Complex call:
storage.EnableReplication(ctx, config, []string{"po", "is", "ea"},
    storage.ReadonlyCells("ix", "gg"),
    storage.OverwritePolicies(true),
    storage.ReplicationInterval(1*time.Hour),
    storage.CopyWorkers(100),
    storage.HealthWatcher(watcher),
)

// Simple call:
storage.EnableReplication(ctx, config, []string{"po", "is", "ea"})
```

### When to use variadic options

> Prefer this option when many of the following apply:
> - Most callers will not need to specify any options.
> - Most options are used infrequently.
> - There are a large number of options.
> - Options require arguments.
> - Options could fail or be set incorrectly (in which case the option function returns an `error`).
> - Options require a lot of documentation that can be hard to fit in a struct.
> - Users or other packages can provide custom options.

### Parameter design

> Options in this style should accept parameters rather than using presence to signal their value; the latter can make dynamic composition of arguments much more difficult.

So:

```go
// Bad:
storage.WithCompression()    // presence = on
storage.NoCompression()      // separate function for off

// Good:
storage.WithCompression(true)
storage.WithCompression(false)
```

> An enumerated option should accept an enumerated constant (e.g. `log.Format(log.Capacitor)` is preferable to `log.CapacitorFormat()`).

### Option processing rules

> In general, options should be processed in order. If there is a conflict or if a non-cumulative option is passed multiple times, the last argument should win.

> The parameter to the option function is generally unexported in this pattern, to restrict the options to being defined only within the package itself.

The unexported-config-struct trick (`replicationOptions` with lowercase `o`) is what prevents third-party packages from defining their own options — the closures need access to unexported fields. To **allow** third-party options, expose the config struct (uppercase) instead.

### Smells (variadic-options anti-patterns)

- Variadic option function with **no** parameters whose presence signals the value (`WithCompression()` for "on") — accept a parameter (`WithCompression(bool)`).
- Two functions where one is a "no-X" variant of another (`WithLogging()` / `WithoutLogging()`) — collapse to `WithLogging(bool)` or use an enum.
- Variadic option pattern used for a function with 2-3 always-required configurations — that's too lightweight to earn the boilerplate; use option struct or positional params.
- Default options not applied (callers must remember to set them every time) — should be merged in via `DefaultXOptions` slice, applied first, then user options.
- Public option type (`type Option = func(*PublicConfig)` with exported `PublicConfig`) when the package didn't intend to allow extension — switch to unexported config to lock down.
- Option function panics on bad input — should return an error if validation can fail (the doc explicitly mentions options that can fail).
- Variadic option processing that doesn't apply "last wins" for non-cumulative options.

---

## Choosing between the two patterns

| Criterion | Option struct | Variadic options |
|---|---|---|
| Most callers need most options | ✅ | ❌ |
| Most callers need few/no options | ❌ | ✅ |
| Few options total (~3-5) | ✅ | ❌ |
| Many options (~10+) | tolerable | ✅ |
| Options frequently used | ✅ | ❌ |
| Options rarely used (long tail) | ❌ | ✅ |
| Options can fail (validation) | awkward | ✅ |
| Third-party packages should define new options | ❌ (struct fields are fixed) | ✅ |
| Need to support multi-parameter options | ❌ | ✅ |
| Want to avoid boilerplate per option | ✅ | ❌ |

When in doubt: **start with option struct**. It's simpler. Migrate to variadic options later if/when the tradeoffs flip.

---

## Consolidated review checklist (2 sub-rules)

### Option struct

- [ ] Functions with 5+ mixed-type parameters considered for an option-struct refactor.
- [ ] No `Ctx context.Context` field on option structs (always positional).
- [ ] Option struct passed by value, not pointer (unless the function mutates).
- [ ] Two sibling functions don't duplicate option-struct types.

### Variadic options

- [ ] Option-function names express **what they do**, not which presence-flag they set (`WithCompression(true)`, not `WithCompression()`).
- [ ] Enumerated options take an enum constant (`log.Format(log.Capacitor)`), not a per-value function.
- [ ] Defaults applied via a `DefaultXOptions` slice processed before user options.
- [ ] Internal config struct unexported when third-party extension is not intended.
- [ ] Options that can fail return an error (or document that they panic).
- [ ] Last-wins semantics applied for non-cumulative options.
- [ ] Doc comments name the default and describe multi-call behaviour ("each call adds" vs "each call overwrites").

## Suggested finding phrasing (advisory)

- "`EnableReplication` has 9 positional parameters — Best Practices §option-structure: extract a `ReplicationOptions` struct as the last (or non-`ctx`) parameter."
- "Option struct `Options` has field `Ctx context.Context` — Best Practices §option-structure: contexts are never included in option structs; keep `ctx` as the first positional parameter."
- "`WithCompression()` (no args) signals \"compression on\" — Best Practices §variadic-options: use parameters (`WithCompression(true)` / `WithCompression(false)`) so dynamic composition works."
- "`log.CapacitorFormat()` and `log.JSONFormat()` are sibling no-arg constructors — Best Practices §variadic-options: prefer an enum-typed option (`log.Format(log.Capacitor)`)."
- "Variadic-option pattern used here but only 3 options exist, all required — Best Practices: option struct earns the boilerplate at this scale; variadic options for the long-tail case."
- "`ApplyOptions` doesn't process `DefaultXOptions` first — Best Practices §variadic-options: defaults should be applied before user options so user options can override."
- "Internal `replicationOptions` is exported (`ReplicationOptions`) — Best Practices §variadic-options: unexport unless third-party packages should be able to define new options."
- "Option `WithRetry(n int)` panics when `n < 0` — Best Practices §variadic-options: options that can fail should return an error rather than panic."

## Sources

- <https://google.github.io/styleguide/go/best-practices#function-argument-lists> — Google Go Best Practices, Function argument lists section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#func-formatting> — Decisions §func-formatting (signatures on one line; sibling skill `decisions-review:05-language.md`)
- <https://google.github.io/styleguide/go/decisions#contexts> — Decisions §contexts (why `ctx` stays positional; `decisions-review:06-common-libraries.md`)
- <https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html> — Rob Pike, *Self-referential functions and the design of options* (the foundational article on the variadic-options pattern)
