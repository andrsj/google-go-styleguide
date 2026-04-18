---
section: "Variable declarations"
number: 6
category: actionable
rules:
  - initialization
  - declaring-with-zero-values
  - composite-literals
  - size-hints
  - channel-direction
sources:
  - https://google.github.io/styleguide/go/best-practices#variable-declarations
---

# Section 06 — Variable declarations (Best Practices)

## What this section covers

Five rules for *how* variables are declared: when to use `:=` vs `var`, when to declare against the zero value vs a composite literal, when (and when not) to add size hints to `make`, and when to constrain channel direction in function signatures.

> [!IMPORTANT]
> These overlap with `decisions-review:05-language.md` (`#zero-value-fields`, `#nil-slices`, `#literal-formatting`). Best Practices clarifies the *declaration* form; Decisions covers the *literal-formatting* form. Together they specify how every variable in a Google Go file should be born.

## #initialization — Initialization

> For consistency, prefer `:=` over `var` when initializing a new variable with a non-zero value.

```go
// Good:
i := 42

// Bad:
var i = 42
```

The reason: `:=` is the unambiguous "declare and initialize" operator. `var i = 42` works but reads as a separate-and-then-assign step that the language doesn't actually do; reserve `var` for the zero-value declarations covered next.

### Smells

- `var x = "foo"` / `var x = 42` / `var x = NewThing()` — switch to `x := ...`.
- `var x int = 42` (with explicit type AND initializer) — usually neither needed; `x := 42` is enough. The explicit type is justified only when the literal's default type isn't what you want (`var x int64 = 42`).
- A single file mixing both patterns for similar declarations.

---

## #declaring-with-zero-values — Declaring variables with zero values

> The following declarations use the zero value:

```go
var (
    coords Point
    magic  [4]byte
    primes []int
)
```

> You should declare values using the zero value when you want to convey an empty value that **is ready for later use**.

> Using composite literals with explicit initialization can be clunky:

```go
// Bad — explicit zero values:
var (
    coords = Point{X: 0, Y: 0}
    magic  = [4]byte{0, 0, 0, 0}
    primes = []int(nil)
)
```

The principle: when the variable starts out empty/zero and will be populated later (appended to, fields set, methods called that mutate it), use the bare `var x T` form. Don't write out the zeros.

This pairs with `decisions-review:05-language.md#nil-slices` (a `nil` slice is functionally equivalent to an empty one for most purposes; prefer `var xs []T` over `[]T{}` or `make([]T, 0)`).

### Smells

- `var x = []T{}` / `var x = make([]T, 0)` — should be `var x []T`.
- `var x = MyStruct{}` (no fields set) — should be `var x MyStruct`.
- `var x = map[K]V(nil)` — should be `var x map[K]V` (and note that a nil map can't be written to without `make`; if you'll write to it, use `make`).
- `var x = [4]byte{0, 0, 0, 0}` — should be `var x [4]byte`.

---

## #composite-literals — Composite literals

> The following are composite literal declarations:

```go
// Good:
var (
    coords   = Point{X: x, Y: y}
    magic    = [4]byte{'I', 'W', 'A', 'D'}
    primes   = []int{2, 3, 5, 7, 11}
    captains = map[string]string{"Kirk": "James Tiberius", "Picard": "Jean-Luc"}
)
```

> You should declare a value using a composite literal when you know initial elements or members.

So the decision is symmetric to the previous rule:

| The variable starts out... | Use |
|---|---|
| Empty and ready for later use | `var x T` (zero-value form) |
| Populated with known initial values | `var x = T{...}` (composite literal form) — or `x := T{...}` for `:=` consistency |
| Populated, but the values aren't known until runtime | `:=` with the runtime expression: `x := compute()` |

(Note: per `#initialization`, `:=` is preferred over `var x = ...` for runtime-known initializers. The `var x = T{...}` form survives mainly for `var (...)` blocks and package-level declarations where `:=` isn't available.)

### Field-name rules carry over

`decisions-review:05-language.md` covers:
- **Field names required for non-local types** (`#field-names`)
- **Repeated type names elidable** (`#repeated-type-names`)
- **Closing braces aligned** (`#literal-matching-braces`)
- **Cuddled braces** (`#cuddled-braces`)

Apply those when writing composite literal declarations.

### Smells

- Composite literal where the type is known but most fields are explicitly zero (`User{Name: "alice", Age: 0, Active: false}`) — drop the zero fields per `decisions-review:05-language.md#zero-value-fields`.
- Field-by-field assignment via `:=` followed by `x.A = 1; x.B = 2` instead of one composite literal — collapse to the literal.
- Composite literal at package-level using `var x = T{...}` while everywhere else in the package uses `:=` — that's fine; `:=` doesn't exist at package level.

---

## #size-hints — Size hints

> The following are declarations that take advantage of size hints in order to preallocate capacity:

```go
// Good:
var (
    // Preferred buffer size for target filesystem: st_blksize.
    buf = make([]byte, 131072)
    // Typically process up to 8-10 elements per run (16 is a safe assumption).
    q = make([]Node, 0, 16)
    // Each shard processes shardSize (typically 32000+) elements.
    seen = make(map[string]bool, shardSize)
)
```

> Size hints and preallocation are important steps **when combined with empirical analysis of the code and its integrations**, to create performance-sensitive and resource-efficient code.

> Most code does not need a size hint or preallocation, and can allow the runtime to grow the slice or map as necessary.

> It is acceptable to preallocate when the final size is known (e.g. when converting between a map and a slice) but this is not a readability requirement, and may not be worth the clutter in small cases.

### Decision tree

| Situation | Pre-size? | Form |
|---|---|---|
| Hot path, profiled, growth dominated allocations | **Yes** — with a comment explaining the magic number | `make([]T, 0, n)` |
| Final size knowable up-front (e.g., converting a map to a slice) | Optional — small ergonomic win | `make([]T, 0, len(src))` |
| Most other code | **No** — let the runtime grow | `var xs []T` (or `:= []T{...}` if known initial elements) |
| Loop building a slice of unknown final size | **No** unless profiled |  `append` away |

Note the difference between **length hint** and **capacity hint** for slices:
- `make([]T, n)` → length n, all zero values, indexable from 0 to n-1.
- `make([]T, 0, n)` → length 0, capacity n, ready to `append` n times without reallocation.

If you mean "I want to `append` n times," use the **capacity** form (`0, n`), not the length form (`n`) — otherwise you'll have n leading zero values + your appended ones.

### Smells

- `make([]T, n)` followed by `append` — accidental zero-padded slice; switch to `make([]T, 0, n)`.
- Size hint with no comment explaining the magic number (e.g., `make([]T, 0, 16)` for unclear reasons).
- Size hint applied "for performance" without profiling — usually premature optimization, drop the hint.
- `make(map[K]V)` (no hint) when a hot path consistently sees thousands of entries — consider adding a hint with a comment.
- `make([]byte, 4096)` for an I/O buffer with no comment about why 4096 — say "matches `io.Copy` default" or `st_blksize` etc.

---

## #channel-direction — Channel direction

> Specify channel direction where possible.

```go
// Good:
// sum computes the sum of all of the values. It reads from the channel until
// the channel is closed.
func sum(values <-chan int) int {
    // ...
}
```

The function signature `<-chan int` (receive-only) makes the contract explicit: this function reads from the channel and won't send or close.

> This prevents casual programming errors that are possible without specification:

```go
// Bad:
func sum(values chan int) (out int) {
    for v := range values {
        out += v
    }
    // values must already be closed for this code to be reachable, which means
    // a second close triggers a panic.
    close(values)
}
```

> When the direction is specified, the compiler catches simple errors like this.

### The three channel direction forms

| Direction in signature | Meaning | Caller's view |
|---|---|---|
| `chan T` | Bidirectional | Function may send, receive, or close |
| `<-chan T` | Receive-only | Function may only receive (read); compiler blocks send/close |
| `chan<- T` | Send-only | Function may only send; compiler blocks receive/close |

### When to use each

- **Producer function** (puts values in a channel) → take `chan<- T` (send-only).
- **Consumer function** (reads values from a channel) → take `<-chan T` (receive-only).
- **Function that does both, or owns the channel and closes it** → take `chan T` (bidirectional).

Bidirectional channels naturally implicit-convert to either send-only or receive-only, so you can call a `<-chan T`-taking function with a `chan T`. The reverse is not true (you can't pass `<-chan T` where `chan T` is expected).

### Smells

- Function signature uses bidirectional `chan T` but the body only sends — should be `chan<- T`.
- Function signature uses bidirectional `chan T` but the body only receives — should be `<-chan T`.
- Function takes `chan T` and calls `close()` from inside — possible double-close bug; clarify channel ownership and consider whether the function should own (and close) or just consume.
- Method on a struct that exposes its internal channel via a getter returning `chan T` instead of `<-chan T` — leaks send/close capability to callers.
- Goroutine-spawning function that doesn't constrain its channel param even though the producer/consumer roles are clear.

---

## Consolidated review checklist (5 sub-rules)

- [ ] `:=` used (not `var x = ...`) when initializing with a non-zero, runtime-known value.
- [ ] Bare `var x T` form used for zero-value declarations that will be populated later (no `var x = T{}` boilerplate).
- [ ] Composite literals (`var x = T{...}` or `x := T{...}`) used when initial values are known.
- [ ] Field-name conventions from `decisions-review:05-language.md` applied (named fields for external types, brace alignment, etc.).
- [ ] `make([]T, 0, n)` for "I want capacity n" — never `make([]T, n)` followed by `append`.
- [ ] Size hints on `make` carry a comment explaining the magic number.
- [ ] Size hints absent in non-hot-path code (let the runtime grow).
- [ ] Channel direction specified in function signatures whenever the role (producer / consumer) is fixed.
- [ ] Bidirectional `chan T` reserved for functions that genuinely own both ends or close the channel.

## Suggested finding phrasing (advisory)

- "`var i = 42` — Best Practices §initialization: prefer `i := 42`."
- "`var xs = []int{}` — Best Practices §declaring-with-zero-values: use the zero-value form `var xs []int` (equivalent for most uses; see also `decisions-review:05-language.md#nil-slices`)."
- "`var u = User{Name: \"alice\", Age: 0, Active: false}` — Best Practices §composite-literals + `decisions-review:05-language.md#zero-value-fields`: drop the zero-value fields (`var u = User{Name: \"alice\"}`)."
- "`buf := make([]byte, 0, n); for ... { buf = append(buf, …) }` is fine, but `buf := make([]byte, n); for ... { buf = append(buf, …) }` would prepend `n` zero bytes; verify the form matches intent."
- "`make([]T, 0, 4096)` with no comment — Best Practices §size-hints: add a comment explaining why 4096 (e.g., \"matches block size\")."
- "`make(map[K]V, 1024)` for a function that processes 5–10 entries — Best Practices §size-hints: drop the hint; small cases don't earn the clutter."
- "`func consume(ch chan int)` body only `range`s — Best Practices §channel-direction: signature should be `<-chan int` so the compiler enforces it."
- "Method `func (s *Server) Output() chan T` exposes bidirectional channel — return `<-chan T` so callers can't send or close."

## Sources

- <https://google.github.io/styleguide/go/best-practices#variable-declarations> — Google Go Best Practices, Variable declarations section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#zero-value-fields> — Decisions §zero-value-fields (sibling skill `decisions-review:05-language.md`)
- <https://google.github.io/styleguide/go/decisions#nil-slices> — Decisions §nil-slices
- <https://google.github.io/styleguide/go/decisions#literal-formatting> — Decisions §literal-formatting
- <https://go.dev/ref/spec#Channel_types> — Go spec on channel types (referenced by `#channel-direction`)
