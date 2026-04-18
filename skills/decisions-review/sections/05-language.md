---
section: "Language"
number: 5
category: actionable
rules:
  - literal-formatting
  - field-names
  - literal-matching-braces
  - cuddled-braces
  - repeated-type-names
  - zero-value-fields
  - nil-slices
  - indentation-confusion
  - func-formatting
  - conditional-formatting
  - copying
  - dont-panic
  - must-functions
  - goroutine-lifetimes
  - interfaces
  - generics
  - pass-values
  - receiver-type
  - switch-and-break
  - synchronous-functions
  - type-aliases
  - use-q
  - use-any
sources:
  - https://google.github.io/styleguide/go/decisions#language
---

# Section 05 — Language

## What this section covers

23 concrete decisions on Go language constructs and idioms — composite literals and brace placement, struct/slice initialization defaults, formatting of functions and conditionals, copying semantics, panics and `Must*` patterns, goroutine lifetimes, interfaces, generics, value vs pointer passing/receivers, switch behaviour, synchronous-first design, type aliases, and the `%q` and `any` idioms.

> [!IMPORTANT]
> This is the largest Decisions section by far. The rules cluster into four themes:
>
> 1. **Literal & formatting** (`#literal-formatting` → `#conditional-formatting`)
> 2. **Memory & lifecycle** (`#copying` → `#goroutine-lifetimes`)
> 3. **Type design** (`#interfaces` → `#receiver-type`)
> 4. **Idioms & sugar** (`#switch-and-break` → `#use-any`)
>
> When dispatching this section's subagent, suggesting these themes can help structure the reasoning.

---

## #literal-formatting — Literal formatting

> Go has an exceptionally powerful composite literal syntax, with which it is possible to express deeply-nested, complicated values in a single expression. Where possible, this literal syntax should be used instead of building values field-by-field.

### Smells

- A series of assignments that builds up a struct/slice/map field-by-field where a single composite literal would do.
- `m := map[string]int{}; m["a"] = 1; m["b"] = 2; …` — collapse to `m := map[string]int{"a": 1, "b": 2}`.

---

## Field names

> Struct literals must specify field names for types defined outside the current package.

```go
// Good:
r := csv.Reader{
    Comma:           ',',
    Comment:         '#',
    FieldsPerRecord: 4,
}
```

> For package-local types, field names are optional.

### Smells

- Positional struct literal `csv.Reader{',', '#', 4, …}` for an external type — must be named.
- Inconsistent style within a single package (some literals named, some positional, for the same local type).

---

## #literal-matching-braces — Matching braces

> The closing half of a brace pair should always appear on a line with the same amount of indentation as the opening brace.

```go
// Good:
good := []*Type{{Key: "value"}}

// Good:
good := []*Type{
    {Key: "multi"},
    {Key: "line"},
}
```

### Smells

- Trailing `}` aligned with the inner content rather than the opening `{`.
- Mixed indentation within one literal where some inner items are on the open line and some on their own lines.

---

## #cuddled-braces — Cuddled braces

> Dropping whitespace between braces (aka "cuddling" them) for slice and array literals is only permitted when both of the following are true: indentation matches **and** inner values are literals or proto builders.

So `[]*Type{{Key: "value"}}` (cuddled `{{`) is OK only when the inner value is itself a literal/builder, not a function call or variable.

### Smells

- `[]*T{NewT(...), NewT(...)}` written as `[]*T{NewT(...){`-shaped cuddle — function-call values can't be cuddled.
- Cuddled braces where the indentation doesn't actually match.

---

## Repeated type names

> Repeated type names may be omitted from slice and map literals. This can be helpful in reducing clutter.

```go
// Verbose (sometimes warranted for complex types):
xs := []*MyLongTypeName{&MyLongTypeName{A: 1}, &MyLongTypeName{B: 2}}

// Cleaner:
xs := []*MyLongTypeName{{A: 1}, {B: 2}}
```

> A reasonable occasion for repeating the type names explicitly is when dealing with a complex type.

### Smells

- Slice/map literal repeating the element type for every entry where the elision form would compile and read more cleanly.
- Elided types in a complex generic literal where readers genuinely need the type spelled out — context-dependent; don't insist.

---

## #zero-value-fields — Zero-value fields

> Zero-value fields may be omitted from struct literals when clarity is not lost as a result.

```go
// Bad — emphasises the irrelevant zero values:
opts := db.Options{Timeout: 5 * time.Second, Retries: 0, MaxConns: 0, Debug: false}

// Good — only the meaningful field:
opts := db.Options{Timeout: 5 * time.Second}
```

The principle: omitting zeros draws attention to what's *actually* being set.

### Smells

- Long struct literal with most fields explicitly set to `0`, `""`, `false`, `nil` — drop them.
- Required-by-API fields set to zero where a non-zero value is the only sensible choice (often a missing config) — flag as a possible bug.

---

## #nil-slices — Nil slices

> For most purposes, there is no functional difference between `nil` and the empty slice.

> If you declare an empty slice as a local variable (especially if it can be the source of a return value), prefer the nil initialization to reduce the risk of bugs by callers.

> Do not create APIs that force their clients to make distinctions between nil and the empty slice.

> When designing interfaces, avoid making a distinction between a `nil` slice and a non-`nil`, zero-length slice, as this can lead to subtle programming errors.

### Smells

- `var xs []T = []T{}` — should be `var xs []T` (the nil form).
- `xs := make([]T, 0)` for a slice that's about to be returned empty — `var xs []T` is idiomatic.
- API contracts that document "returns `nil` if X, empty slice if Y" — collapse both into one return.
- Caller code with `if xs == nil { ... } else if len(xs) == 0 { ... }` — distinction is a bug magnet.

---

## #indentation-confusion — Indentation confusion

> Avoid introducing a line break if it would align the rest of the line with an indented code block.

> If this is unavoidable, leave a space to separate the code in the block from the wrapped line.

The classic case:

```go
// Bad — wrapped condition aligns with indented body:
if veryLongConditionA &&
    veryLongConditionB {
    body()
}

// Good — extract or restructure:
ok := veryLongConditionA && veryLongConditionB
if ok {
    body()
}
```

### Smells

- Wrapped expressions inside `if` / `for` / `switch` headers that visually merge with the body.
- Multi-line function arguments that line up with the function body's indentation.

---

## #func-formatting — Function formatting

> The signature of a function or method declaration should remain on a single line to avoid indentation confusion.

> Function argument lists can make some of the longest lines in a Go source file.

> Lines can often be shortened by factoring out local variables.

> Similarly, function and method calls should not be separated based solely on line length.

### Smells

- Function signature wrapped before the opening `{`.
- Function call wrapped purely to fit width — refactor by extracting variables, not wrapping.
- Ten-parameter signatures — fewer parameters or an Options struct, not multi-line.

(See also Style Guide `#08 Line length`.)

---

## #conditional-formatting — Conditionals and loops

> An `if` statement should not be line broken; multi-line `if` clauses can lead to indentation confusion.

> If the short-circuit behavior is not required, the boolean operands can be extracted directly.

> `switch` and `case` statements should also remain on a single line.

### Smells

- `if a && b && c {` wrapped across lines — extract `cond := a && b && c`.
- `case longExpr1, longExpr2,\n longExpr3:` wrapped — usually means the case list is too complex; refactor.
- Wrapped `for` headers (`for i := 0; longCondition; i++` split across lines).

---

## #copying — Copying

> To avoid unexpected aliasing and similar bugs, be careful when copying a struct from another package. For example, synchronization objects such as `sync.Mutex` must not be copied.

> The `bytes.Buffer` type contains a `[]byte` slice and, as an optimization for small strings, a small byte array to which the slice may refer. If you copy a `Buffer`, the slice in the copy may alias the array in the original, causing subsequent method calls to have surprising effects.

> In general, do not copy a value of type `T` if its methods are associated with the pointer type, `*T`.

```go
// Bad:
b1 := bytes.Buffer{}
b2 := b1
```

> Invoking a method that takes a value receiver can hide the copy. When you author an API, you should generally take and return pointer types if your structs contain fields that should not be copied.

> This guidance also applies to copying `sync.Mutex`.

### Smells

- `var b sync.Mutex; b2 := b` — never; always pointer.
- `bytes.Buffer`, `strings.Builder`, `sync.WaitGroup`, `sync.Mutex`, `sync.Once`, `sync.Cond` passed/returned by value.
- Type's methods are all `func (t *T) ...` but `T` is also passed by value somewhere (and `vet` `copylocks` would complain).
- Constructor returns the type by value when its methods are pointer-only — return `*T` instead.

(`go vet`'s `copylocks` checker catches some of these mechanically.)

---

## #dont-panic — Don't panic

> Do not use `panic` for normal error handling. Instead, use `error` and multiple return values.

> Within `package main` and initialization code, consider `log.Exit` for errors that should terminate the program (e.g., invalid configuration), as in many of these cases a stack trace will not help the reader.

> For errors that indicate "impossible" conditions, namely bugs that should always be caught during code review and/or testing, a function may reasonably return an error or call `log.Fatal`.

### Decision tree

| Situation | Use |
|---|---|
| Normal error | return `error`, never `panic` |
| Invalid config / startup failure | `log.Exit` (no stack) |
| "Impossible" bug — invariant violated | return `error` or `log.Fatal` (with stack) |
| Library code | never `panic` for control flow |

### Smells

- `panic(err)` in non-test, non-init code.
- Library function panics on bad input instead of returning an error.
- `panic("not implemented")` in shipped code (TODO never resolved).
- `recover()` outside of an explicit defensive boundary (HTTP middleware, RPC server) — usually a sign of a panic that should be an error.

---

## #must-functions — Must functions

> Setup helper functions that stop the program on failure follow the naming convention `MustXYZ` (or `mustXYZ`). In general, they should only be called early on program startup, not on things like user input where normal Go error handling is preferred.

> This often comes up for functions called to initialize package-level variables exclusively at package initialization time (e.g. `template.Must` and `regexp.MustCompile`).

```go
var DefaultVersion = MustParse("1.2.3")
```

> The same convention may be used in test helpers that only stop the current test (using `t.Fatal`).

> In both of these cases, the value of this pattern is that the helpers can be called in a "value" context. These helpers should not be called in places where it's difficult to ensure an error would be caught or in a context where an error should be checked (e.g., in many request handlers).

> They should not be used when ordinary error handling is possible.

### Smells

- `MustParse`, `MustCompile`, `MustNew` called inside a request handler or per-request code path — switch to error-returning variant.
- `MustX` helper that doesn't actually `panic`/`log.Fatal` on failure — naming lies.
- A new `MustX` defined when `template.Must` / `regexp.MustCompile` already exists — reuse stdlib.
- `MustX` in production library code where the caller should decide how to handle the error.

---

## #goroutine-lifetimes — Goroutine lifetimes

> When you spawn goroutines, make it clear when or whether they exit.

> Goroutines can leak by blocking on channel sends or receives. The garbage collector will not terminate a goroutine blocked on a channel even if no other goroutine has a reference to the channel.

> Even when goroutines do not leak, leaving them in-flight when they are no longer needed can cause other subtle and hard-to-diagnose problems. Sending on a channel that has been closed causes a panic.

```go
// Bad:
ch := make(chan int)
ch <- 42
close(ch)
ch <- 13 // panic
```

> Modifying still-in-use inputs "after the result isn't needed" can lead to data races. Leaving goroutines in-flight for arbitrarily long can lead to unpredictable memory usage.

> Concurrent code should be written such that the goroutine lifetimes are obvious.

> Code that follows best practices around context usage often helps make this clear. It is conventionally managed with a `context.Context`.

### Smells

- `go func() { … }()` with no clear cancellation mechanism (no context, no stop channel, no `WaitGroup`).
- Send to a channel that may already be closed without a `select` + `done` channel guard.
- Goroutine that captures a loop variable directly — classic data race / wrong-value bug.
- Goroutines that outlive the request that spawned them with no `context.Context` cancellation.
- A `defer wg.Done()` placed after a possible early return inside the goroutine — leaks.

---

## #interfaces — Interfaces

> Avoid creating interfaces until a real need exists. Focus on the required behavior rather than just abstract named patterns like "service" or "repository" and the like.

> Do not wrap RPC clients in new manual interfaces just for the sake of abstraction or testing.

> Do not define back doors or export test double implementations of an interface solely for testing.

> Design interfaces to be small for easier implementation and composition. Document interfaces appropriately including their contract, edge cases, and expected errors.

> The consumer of the interface should define it (not the package implementing the interface), ensuring it includes only the methods they actually use.

> Functions should take interfaces as arguments but return concrete types. Returning concrete types allows the caller to have access to every public method and field of that specific implementation.

> Sometimes returning an interface is acceptable for encapsulation (e.g., `error` interface), and certain constructs like command, chaining, factory, and strategy patterns.

### Smells

- "Just-in-case" interfaces with one implementation and no real consumer pressure.
- Interfaces named `XService`, `XRepository`, `XManager` mirroring the struct without behaviour-driven justification.
- Hand-written `MockX` interface wrapping an RPC client purely for testing.
- Producer-side interfaces (defined in the package that implements them, in case "future implementations" want to plug in).
- Function returning `interface{}` or a custom interface where the concrete type would expose useful methods to callers.
- 10-method interfaces — too large; split or compose.

---

## #generics — Generics

> Generics (formally called "Type Parameters") are allowed where they fulfill your business requirements.

> In many applications, a conventional approach using existing language features (slices, maps, interfaces, and so on) works just as well without the added complexity, so be wary of premature use.

> When introducing an exported API that uses generics, make sure it is suitably documented.

> Do not use generics just because you are implementing an algorithm or data structure that does not care about the type of its member elements.

> If there is only one type being instantiated in practice, start by making your code work on that type without using generics at all.

> Do not use generics to invent domain-specific languages (DSLs).

> Refrain from introducing error-handling frameworks that might put a significant burden on readers.

### Smells

- Generic function instantiated in only one type at every call site — drop the type parameter.
- A "generic util" package wrapping `slices` / `maps` operations the stdlib already has (Go ≥ 1.21).
- Generic builders / fluent APIs encoded in type parameters — the "DSL" anti-pattern the doc explicitly bans.
- Generic error-handling pattern (e.g., `Result[T, E]`) — the "error-handling framework" anti-pattern.
- Multi-constraint generics where the constraint nearly equals one concrete type.

---

## #pass-values — Pass values

> Do not pass pointers as function arguments just to save a few bytes.

> If a function reads its argument `x` only as `*x` throughout, then the argument shouldn't be a pointer.

> Common instances of this include passing a pointer to a string (`*string`) or a pointer to an interface value (`*io.Reader`). In both cases, the value itself is a fixed size and can be passed directly.

> This advice does not apply to large structs, or even small structs that may increase in size.

> Protocol buffer messages should generally be handled by pointer rather than by value.

### Smells

- `func Foo(s *string)` where `s` is only ever dereferenced — change to `func Foo(s string)`.
- `*io.Reader`, `*int`, `*bool` parameters where the value is read but not assigned through.
- Parameter passed by value where the struct is genuinely large (think >100 bytes) — inverse smell, switch to pointer.
- `*proto.Message` (or generated proto type) passed by value — protos are pointer-conventional.

---

## #receiver-type — Receiver type

> A method receiver can be passed either as a value or a pointer, just as if it were a regular function parameter.

> The choice between the two is based on which method sets the method should be a part of.

> Correctness wins over speed or simplicity.

Concrete rules from the doc:

- "If the receiver is a slice and the method doesn't reslice or reallocate the slice, use a value rather than a pointer."
- "If the method needs to mutate the receiver, the receiver must be a pointer."
- "If the receiver is a struct containing fields that cannot safely be copied, use a pointer receiver."
- "If the receiver is a 'large' struct or array, a pointer receiver may be more efficient."
- "If the receiver is a map, function, or channel, use a value rather than a pointer."
- "When in doubt, use a pointer receiver."
- "As a general guideline, prefer to make the methods for a type either all pointer methods or all value methods."

### Smells

- Same type with mixed pointer/value receivers across methods (`func (t T) A()` and `func (t *T) B()`) — pick one.
- Pointer receiver on a tiny non-mutating method (`func (s *Status) String() string` where `Status` is an int alias) — `func (s Status) String() string`.
- Value receiver on a struct containing `sync.Mutex` — must be pointer (also see `#copying`).
- Pointer receiver on a `map`/`chan`/`func` underlying type — use value.

---

## #switch-and-break — Switch and break

> Do not use `break` statements without target labels at the ends of `switch` clauses; they are redundant.

`switch` clauses in Go automatically break, and a `fallthrough` statement is needed to achieve the C-style behavior. It recommends using a comment rather than `break` if you want to clarify the purpose of an empty clause.

```go
// Good:
switch x {
case "A", "B":
    buf.WriteString(x)
case "C":
    // handled outside of the switch statement
default:
    return fmt.Errorf("unknown value: %q", x)
}
```

```go
// Bad:
switch x {
case "A", "B":
    buf.WriteString(x)
    break // this break is redundant
case "C":
    break // this break is redundant
default:
    return fmt.Errorf("unknown value: %q", x)
}
```

### Smells

- Trailing `break` in any `case` clause without a target label.
- `break` used "for clarity" — replace with a comment.
- Labeled `break` (`break outer`) is fine and not what this rule targets — only the label-less, end-of-case `break`.

---

## #synchronous-functions — Synchronous functions

> Synchronous functions return their results directly and finish any callbacks or channel operations before returning.

> Prefer synchronous functions over asynchronous functions.

> Synchronous functions keep goroutines localized within a call. This helps to reason about their lifetimes, and avoid leaks and data races.

> Synchronous functions are also easier to test, since the caller can pass an input and check the output without the need for polling or synchronization.

> If necessary, the caller can add concurrency by calling the function in a separate goroutine. However, it is quite difficult (sometimes impossible) to remove unnecessary concurrency at the caller side.

### Smells

- Function spawns a goroutine internally and returns a channel/future the caller must drain.
- Function takes a callback for the "result" instead of returning it.
- Function returns immediately and does work in a background goroutine — the result lifetime is now caller-unclear.
- API has both `Foo` and `FooAsync` variants — usually means the sync one should be the only one.

---

## #type-aliases — Type aliases

> Use a *type definition*, `type T1 T2`, to define a new type.

> Use a *type alias*, `type T1 = T2`, to refer to an existing type without defining a new type.

> Type aliases are rare; their primary use is to aid migrating packages to new source code locations. Don't use type aliasing when it is not needed.

### Smells

- Type alias used as a "shorter name" for an existing type within a single package — usually doesn't earn its keep.
- Type alias to wrap an external type "for branding" — should be a type definition (`type T1 T2`), not an alias (`type T1 = T2`), since the goal is a distinct type.
- Type alias remaining after a migration is complete — remove.
- Confusion between `type T1 = T2` and `type T1 T2` — different semantics; alias `=` does NOT create a new type.

---

## #use-q — Use %q

> Go's format functions (`fmt.Printf` etc.) have a `%q` verb which prints strings inside double-quotation marks.

```go
// Good:
fmt.Printf("value %q looks like English text", someText)
```

```go
// Bad:
fmt.Printf("value \"%s\" looks like English text", someText)
// Avoid manually wrapping strings with single-quotes too:
fmt.Printf("value '%s' looks like English text", someText)
```

> Using `%q` is recommended in output intended for humans where the input value could possibly be empty or contain control characters. It can be very hard to notice a silent empty string, but `""` stands out clearly as such.

### Smells

- `fmt.Sprintf("\"%s\"", x)` — replace with `%q`.
- `fmt.Sprintf("'%s'", x)` (single quotes manually) — replace with `%q`.
- `log.Printf("got %s", x)` where `x` could be empty / contain newlines — `%q` makes the failure visible.

---

## #use-any — Use any

> Go 1.18 introduces an `any` type as an alias to `interface{}`.

> Because it is an alias, `any` is equivalent to `interface{}` in many situations and in others it is easily interchangeable via an explicit conversion.

> Prefer to use `any` in new code.

### Smells

- `interface{}` in new code (Go ≥ 1.18 codebases) — switch to `any`.
- Mixed `interface{}` and `any` within the same package — pick one.
- (Counter-smell: an `interface{}` in code targeting a Go version <1.18 — leave it; the rule is for new Go.)

---

## Consolidated review checklist (all 23 rules)

### Literal & formatting

- [ ] Composite literals used over field-by-field assignment.
- [ ] Field names specified for struct literals of types from other packages.
- [ ] Closing braces aligned with opening braces.
- [ ] Cuddled braces only when indentation matches and inner values are literals/builders.
- [ ] Repeated type names elided in slice/map literals (unless the type is genuinely complex).
- [ ] Zero-value fields omitted from struct literals.
- [ ] Empty slices declared as `var xs []T`, not `[]T{}` or `make([]T, 0)`.
- [ ] No multi-line `if` / `switch` / `for` headers; conditions extracted to `:=` variables.
- [ ] Function signatures on one line; calls not split for line-length alone.
- [ ] No "indentation confusion" (wrapped expressions visually merging with body).

### Memory & lifecycle

- [ ] Sync primitives, `bytes.Buffer`, `strings.Builder` always handled by pointer.
- [ ] Methods on `T` are not all-`*T`-only with `T` also being copied somewhere.
- [ ] No `panic` for normal error handling.
- [ ] `MustX` patterns confined to package init / tests.
- [ ] Every spawned goroutine has a clear exit story (context, WaitGroup, done channel).
- [ ] No sends on potentially-closed channels without guards.

### Type design

- [ ] Interfaces created only when a real consumer needs them; defined consumer-side.
- [ ] Functions take interfaces, return concrete types (with stated exceptions).
- [ ] Generics introduced only where business requirements demand them.
- [ ] No generic DSLs or generic error-handling frameworks.
- [ ] Pointers passed only where needed (large struct, mutation, proto).
- [ ] Receiver types chosen by the rules (mutation → pointer; map/chan/func → value; consistent across the type).

### Idioms & sugar

- [ ] No redundant `break` at end of switch cases.
- [ ] Synchronous design preferred; async wrappers added by callers, not pre-baked.
- [ ] Type aliases (`type T1 = T2`) reserved for migrations; type definitions (`type T1 T2`) used for new types.
- [ ] `%q` used for human-output strings instead of manually quoting `"%s"`.
- [ ] `any` used instead of `interface{}` in new code.

## Suggested finding phrasing

- "`opts := db.Options{Timeout: 5 * time.Second, Retries: 0, MaxConns: 0}` — Decisions §zero-value-fields: drop the zero-valued fields."
- "`var xs []T = []T{}` — Decisions §nil-slices: use `var xs []T` (the nil form is idiomatic and equivalent)."
- "`if a && b && c {` wrapped across lines — Decisions §conditional-formatting: extract `cond := a && b && c` and check `if cond {`."
- "`b1 := bytes.Buffer{}; b2 := b1` — Decisions §copying: `bytes.Buffer` must not be copied; pass `*bytes.Buffer`."
- "`panic(err)` in `internal/parser.Parse` — Decisions §dont-panic: return `error` instead."
- "`MustParse(userInput)` in HTTP handler — Decisions §must-functions: `MustX` is for init/tests; use the error-returning variant in request paths."
- "`go func() { for { … } }()` with no context cancellation — Decisions §goroutine-lifetimes: every spawned goroutine needs a clear exit; thread `ctx context.Context` through and `select` on `<-ctx.Done()`."
- "Interface `UserServiceInterface` defined in `userservice` package with one implementation — Decisions §interfaces: don't pre-declare; define consumer-side when a real need arises."
- "Generic `Result[T, E]` type used for error handling — Decisions §generics: do not introduce error-handling frameworks via generics."
- "`func Foo(s *string)` where `s` is only ever dereferenced — Decisions §pass-values: take the value, not the pointer."
- "Mixed receiver types: `func (s Server) A()` and `func (s *Server) B()` — Decisions §receiver-type: choose one (almost always pointer here) and apply to all methods."
- "Trailing `break` after `buf.WriteString(x)` in case clause — Decisions §switch-and-break: redundant; remove."
- "`func StartProcessing(...) <-chan Result` returns a channel and starts background work — Decisions §synchronous-functions: prefer synchronous; let callers add concurrency."
- "`type UserID = int64` (alias) for a type that should be distinct — Decisions §type-aliases: use type definition `type UserID int64` instead."
- "`fmt.Printf(\"got \\\"%s\\\"\", name)` — Decisions §use-q: use `%q` (`fmt.Printf(\"got %q\", name)`)."
- "`map[string]interface{}` in new code — Decisions §use-any: use `map[string]any`."

## Sources

- <https://google.github.io/styleguide/go/decisions#language> — Google Go Style Decisions, Language section (the canonical text quoted above)
- <https://golang.org/ref/spec#The_zero_value> — Go spec on zero values (`#zero-value-fields`)
- <https://pkg.go.dev/text/template#Must> — `template.Must` (`#must-functions`)
- <https://pkg.go.dev/regexp#MustCompile> — `regexp.MustCompile` (`#must-functions`)
- <https://pkg.go.dev/context> — `context.Context` (`#goroutine-lifetimes`)
- <https://pkg.go.dev/cmd/vet> — `go vet` (`copylocks` checker covers some `#copying` violations mechanically)
