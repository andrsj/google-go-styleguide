---
section: "Naming"
number: 1
category: actionable
rules:
  - function-and-method-names (avoid-repetition, naming-conventions)
  - test-double-and-helper-packages (creating-test-helpers, simple-case, multiple-behaviors, multiple-doubles, local-variables-in-tests)
  - shadowing
  - util-packages
sources:
  - https://google.github.io/styleguide/go/best-practices#naming
---

# Section 01 — Naming (Best Practices)

## What this section covers

Pragmatic guidance on naming functions, methods, test doubles, and packages. These are **suggestions**, not the hard rules in `guide-review:#09 Naming` or the specific decisions in `decisions-review:01-naming.md`. Use them as judgment-shapers, not findings to enforce mechanically.

> [!IMPORTANT]
> Where this section conflicts with `guide-review` or `decisions-review`, those documents win. The Style Guide's `#09 Naming` (no repetition, context-aware) and the Decisions' detailed naming rules are the source of truth; Best Practices fills the gaps with patterns.

## Function and method names

### Avoid repetition

> When choosing the name for a function or method, consider the context in which the name will be read.

You can typically omit:

- The types of the inputs and outputs (when there is no collision)
- The type of a method's receiver
- Whether an input or output is a pointer

#### For functions: don't repeat the package name

```go
// Bad:
package yamlconfig
func ParseYAMLConfig(input string) (*Config, error)

// Good:
package yamlconfig
func Parse(input string) (*Config, error)
```

#### For methods: don't repeat the receiver type

```go
// Bad:
func (c *Config) WriteConfigTo(w io.Writer) (int64, error)

// Good:
func (c *Config) WriteTo(w io.Writer) (int64, error)
```

#### For parameters: don't repeat names of variables passed in

```go
// Bad:
func OverrideFirstWithSecond(dest, source *Config) error

// Good:
func Override(dest, source *Config) error
```

#### For return values: don't repeat the names and types

```go
// Bad:
func TransformToJSON(input *Config) *jsonconfig.Config

// Good:
func Transform(input *Config) *jsonconfig.Config
```

#### Disambiguation is OK

> When it is necessary to disambiguate functions of a similar name, it is acceptable to include extra information.

```go
// Good:
func (c *Config) WriteTextTo(w io.Writer) (int64, error)
func (c *Config) WriteBinaryTo(w io.Writer) (int64, error)
```

### Naming conventions

#### Noun-like names for functions that return

> Functions that return something are given noun-like names.

```go
// Good:
func (c *Config) JobName(key string) (value string, ok bool)
```

> A corollary of this is that function and method names should avoid the prefix `Get`. (See also `decisions-review:01-naming.md#getters`.)

```go
// Bad:
func (c *Config) GetJobName(key string) (value string, ok bool)
```

#### Verb-like names for functions that do

> Functions that do something are given verb-like names.

```go
// Good:
func (c *Config) WriteDetail(w io.Writer) (int64, error)
```

#### Type differentiation

> Identical functions that differ only by the types involved include the name of the type at the end of the name.

```go
// Good:
func ParseInt(input string) (int, error)
func ParseInt64(input string) (int64, error)
func AppendInt(buf []byte, value int) []byte
func AppendInt64(buf []byte, value int64) []byte
```

> If there is a clear "primary" version, the type can be omitted from the name for that version:

```go
// Good:
func (c *Config) Marshal() ([]byte, error)
func (c *Config) MarshalText() (string, error)
```

### Smells

- `package foo; func ParseFoo(...)` — drop the suffix.
- `func (c *Config) ConfigSomething(...)` — drop the receiver type.
- `func OverrideFirstWithSecond(dest, source ...)` — name parameters in the signature; the function name shouldn't.
- `Get`-prefixed accessors that just return a field.
- Identical-shape functions for `int` / `int64` / `string` differing only in suffix when no clear primary exists — that's *fine* per this rule; only flag if the suffix is missing.

---

## Test double and helper packages

> There are several disciplines you can apply to naming packages and types that provide test helpers and especially test doubles.

### Creating test helper packages

Suppose you have a focused production package:

```go
package creditcard

import (
    "errors"
    "path/to/money"
)

var ErrDeclined = errors.New("creditcard: declined")

type Card struct{}
type Service struct{}
func (s *Service) Charge(c *Card, amount money.Money) error { /* omitted */ }
```

> A safe choice is to append the word `test` to the original package name.

```go
// Good:
package creditcardtest
```

### Simple case (one type to double)

> If you anticipate only test doubles for one type (like `Service`), you can take a concise approach to naming the doubles:

```go
// Good:
import (
    "path/to/creditcard"
    "path/to/money"
)

type Stub struct{}

func (Stub) Charge(*creditcard.Card, money.Money) error { return nil }
```

> This is strictly preferable to a naming choice like `StubService` or the very poor `StubCreditCardService`.

If using Bazel:

```python
go_library(
    name = "creditcardtest",
    srcs = ["creditcardtest.go"],
    deps = [":creditcard", ":money"],
    testonly = True,
)
```

### Multiple test double behaviors

> When one kind of stub is not enough (for example, you also need one that always fails), we recommend naming the stubs according to the behavior they emulate.

```go
// Good:
type AlwaysCharges struct{}
func (AlwaysCharges) Charge(*creditcard.Card, money.Money) error { return nil }

type AlwaysDeclines struct{}
func (AlwaysDeclines) Charge(*creditcard.Card, money.Money) error { return creditcard.ErrDeclined }
```

### Multiple doubles for multiple types

When the package has multiple types worth doubling:

```go
// Good:
type StubService struct{}
func (StubService) Charge(*creditcard.Card, money.Money) error { return nil }

type StubStoredValue struct{}
func (StubStoredValue) Credit(*creditcard.Card, money.Money) error { return nil }
```

### Local variables in tests

> When variables in your tests refer to doubles, choose a name that most clearly differentiates the double from other production types.

The recommended pattern: prefix the local variable so it's visually distinct from production-type names:

```go
// Good:
var spyCC creditcardtest.Spy
proc := &Processor{CC: spyCC}
```

```go
// Bad — naming collision risk with production type CC:
var cc creditcardtest.Spy
proc := &Processor{CC: cc}
```

The `spyCC` form makes the test reader notice immediately that `spyCC` is the double, not the production object.

### Smells

- Test helper package named `<pkg>_helpers`, `<pkg>_test_helpers`, `<pkg>_mocks`, `<pkg>_fakes` — prefer `<pkg>test` (e.g. `creditcardtest`).
- A single-type stub named with the type appended (`StubService`, `StubCreditCardService`) when there's only one type to double — use plain `Stub`.
- Stubs for multiple behaviors named generically (`Stub1`, `Stub2`) — name by behavior (`AlwaysCharges`, `AlwaysDeclines`).
- Local test-double variable named the same as the production-side field (`cc` vs production `CC`) — prefix with `spy`/`stub`/`fake`/`mock` to disambiguate.
- Bazel `go_library` for a `*test` package missing `testonly = True`.

---

## Shadowing

> Like many programming languages, Go has mutable variables: assigning to a variable changes its value.

Two distinct concepts:

- **Stomping**: short-variable-declaration (`:=`) reassigns an existing variable in scope when the original value is no longer needed. Acceptable.
- **Shadowing**: `:=` inside a nested scope creates a *new* variable that hides the outer one. Code outside the inner block keeps seeing the outer value. This is a common bug source.

### Bad — accidental shadowing

```go
func (s *Server) innerHandler(ctx context.Context, req *pb.MyRequest) *pb.MyResponse {
    if *shortenDeadlines {
        ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
        defer cancel()
        ctxlog.Info(ctx, "Capped deadline in inner request")
    }
    // BUG: "ctx" here again means the context that the caller provided.
}
```

The `ctx, cancel := …` inside the `if` block shadows the outer `ctx`. Any code after the `if` uses the un-shortened context.

### Good — explicit reassignment

```go
func (s *Server) innerHandler(ctx context.Context, req *pb.MyRequest) *pb.MyResponse {
    if *shortenDeadlines {
        var cancel func()
        ctx, cancel = context.WithTimeout(ctx, 3*time.Second)
        defer cancel()
        ctxlog.Info(ctx, "Capped deadline in inner request")
    }
    // ctx here is the shortened one.
}
```

The `var cancel func()` declaration combined with `=` (not `:=`) reuses the outer `ctx`.

### Don't shadow standard package names

> It is not a good idea to use variables with the same name as standard packages other than very small scopes, because that renders free functions and values from that package inaccessible.

```go
// Bad:
url := req.URL.String()
// In this scope, you can no longer call url.Parse(...) — the package is shadowed.

// Good:
u := req.URL.String()
```

Common collisions: `time`, `url`, `path`, `os`, `io`, `log`, `context`, `fmt`, `errors`.

### Smells

- `:=` inside an `if` / `for` / `switch` block re-declaring a variable that's also used after the block (especially `ctx`, `err`, `cancel`).
- `tx, err := db.Begin()` inside a deferred callback that hides an outer `err` the caller is checking.
- Local variables named `time`, `url`, `path`, `log`, `os`, etc. — the package they shadow becomes inaccessible in scope.
- Shadowed `err` inside `defer func() { … }()` — the outer `err` won't be modified.
- Shadowing pattern in test loops: `for _, tt := range tests { tt := tt; t.Run(tt.name, func(t *testing.T) { … }) }` — this *intentional* shadow is required for `t.Parallel()`; not a smell.

(`go vet` has a `-shadow` checker — disabled by default but useful to enable.)

---

## Util packages

> Go package names should be related to what the package provides. Naming a package just `util`, `helper`, `common` or similar is usually a poor choice.

> Uninformative names make the code harder to read, and if used too broadly they are liable to cause needless import conflicts.

### Bad — readers can't tell what each call does

```go
db := test.NewDatabaseFromFile(...)
_, err := f.Seek(0, common.SeekStart)
b := helper.Marshal(curve, x, y)
```

### Good — package name carries domain meaning

```go
db := spannertest.NewDatabaseFromFile(...)
_, err := f.Seek(0, io.SeekStart)
b := elliptic.Marshal(curve, x, y)
```

You can tell roughly what each of these does even without knowing the imports list.

### Smells

- Package literally named `util`, `helper`, `helpers`, `common`, `lib`, `tools`, `misc`, `shared`.
- Sub-packages like `internal/util`, `pkg/common` — same anti-pattern.
- A `util` package that's grown to >5 unrelated functions — split by domain (parse-related → `parsing`; time-related → `timeutil`; etc.).
- Two different `util` packages imported in the same file — high chance one needs an alias.
- A package named after the team or component owner (`teamfoo_util`) — name it for what it provides.

### When a generic name is OK

- Tiny stdlib-shaped utility wrappers around a single concept (`stringutil`, `sliceutil`, `timeutil`) where the prefix carries information. Better than bare `util`.
- Internal-only test helpers — see the `<pkg>test` convention above.

---

## Consolidated review checklist (4 sub-rules)

- [ ] Function/method names don't repeat the package, receiver type, parameter names, or return types.
- [ ] Functions returning values use noun-like names; functions doing work use verb-like names.
- [ ] No `Get` prefix on field-returning accessors.
- [ ] Type-differentiated functions name the type as a suffix (`ParseInt`, `ParseInt64`); the "primary" version may omit the suffix.
- [ ] Test helper packages named `<pkg>test`, not `<pkg>_helpers` / `_mocks` / `_fakes`.
- [ ] Single-type test double named `Stub` (not `StubService`); multi-type test doubles named with the type (`StubService`, `StubStoredValue`); behavior-differentiated doubles named by behavior (`AlwaysCharges`).
- [ ] Local test-double variables prefixed (`spyCC`, not `cc`) to differentiate from production types.
- [ ] No `:=` shadowing of variables used after the inner scope (especially `ctx`, `err`, `cancel`).
- [ ] No local variables shadowing standard package names (`time`, `url`, `path`, `log`, `os`, etc.).
- [ ] No package named `util`, `helper`, `common`, `lib`, `tools`, `misc`, `shared` — name it for what it provides.

## Suggested finding phrasing (advisory)

- "`yamlconfig.ParseYAMLConfig` repeats the package name — Best Practices §function-and-method-names: consider `yamlconfig.Parse`."
- "`func (c *Config) WriteConfigTo(w)` — Best Practices: drop the receiver type from the method name; `WriteTo` reads cleaner."
- "Test helper package `creditcard_helpers` — Best Practices recommends the `<pkg>test` convention: rename to `creditcardtest`."
- "Test double named `StubService` for the only doubled type — Best Practices recommends just `Stub` when there's a single type."
- "Local var `cc creditcardtest.Spy` collides visually with production field `CC` — Best Practices: prefix the test-double variable (`spyCC`)."
- "`ctx, cancel := context.WithTimeout(ctx, ...)` inside `if` block, `ctx` used afterward — Best Practices §shadowing: declare `cancel` separately and reassign `ctx` with `=` to avoid shadowing the outer context."
- "Local variable `url` shadows the `net/url` package — Best Practices §shadowing: rename to `u` or similar."
- "Package named `util` — Best Practices §util-packages: rename to something domain-specific (`stringutil`, `parsing`, etc.)."

## Sources

- <https://google.github.io/styleguide/go/best-practices#naming> — Google Go Best Practices, Naming section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#naming> — Decisions, expanded naming guidance (sibling skill `decisions-review:01-naming.md`)
- <https://google.github.io/styleguide/go/guide#naming> — Style Guide, Naming principles (sibling skill `guide-review:09-naming.md`)
- <https://pkg.go.dev/cmd/vet> — `go vet` (`-shadow` checker covers some `#shadowing` violations)
