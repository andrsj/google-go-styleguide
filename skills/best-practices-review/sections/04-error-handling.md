---
section: "Error handling"
number: 4
category: actionable
rules:
  - error-structure
  - adding-information-to-errors
  - placement-of-w-in-errors (sentinel-error-placement)
  - logging-errors (custom-verbosity-levels)
  - program-initialization
  - program-checks-and-panics
  - when-to-panic
sources:
  - https://google.github.io/styleguide/go/best-practices#error-handling
---

# Section 04 — Error handling (Best Practices)

## What this section covers

Seven recommendations on how to *structure* errors (not just whether to return them — that's covered by `decisions-review:04-errors.md`). Topics: programmatic error inspection via `errors.Is`/`errors.As`, what context to add when wrapping, where to put `%w`, when to log, when to use `log.Exit` vs `log.Fatal` vs `panic`.

> [!IMPORTANT]
> This section depends on the *foundational* error rules from `decisions-review:04-errors.md` (return `error`, lowercase strings, no in-band errors, early returns). Best Practices then adds the *structural* layer: how to make errors interrogable, wrap them well, and place sentinels.

## #error-structure — Error structure

> If callers need to interrogate the error (e.g., distinguish different error conditions), give the error value structure so that this can be done programmatically rather than having the caller perform string matching.

### Good — sentinel errors + `errors.Is`

```go
type Animal string

var (
    ErrDuplicate = errors.New("duplicate")
    ErrMarsupial = errors.New("marsupials are not supported")
)

func process(animal Animal) error {
    switch {
    case seen[animal]:
        return ErrDuplicate
    case marsupial(animal):
        return ErrMarsupial
    }
    seen[animal] = true
    return nil
}
```

> The caller can simply compare the returned error value of the function with one of the known error values.

> That is perfectly adequate in many cases. If `process` returns wrapped errors (discussed below), you can use `errors.Is`.

```go
func handlePet(...) {
    switch err := process(an); {
    case errors.Is(err, ErrDuplicate):
        return fmt.Errorf("feed %q: %v", an, err)
    case errors.Is(err, ErrMarsupial):
        // ...
    }
}
```

### Bad — string matching

> Do not attempt to distinguish errors based on their string form.

```go
// Bad:
func handlePet(...) {
    err := process(an)
    if regexp.MatchString(`duplicate`, err.Error()) {...}
    if regexp.MatchString(`marsupial`, err.Error()) {...}
}
```

> If there is extra information in the error that the caller needs programmatically, it should ideally be presented structurally.

For richer carry-along data, define a custom error type with fields, and use `errors.As`:

```go
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %q not found", e.Resource, e.ID)
}

// Caller:
var nfe *NotFoundError
if errors.As(err, &nfe) {
    log.Printf("missing %s id=%s", nfe.Resource, nfe.ID)
}
```

### Smells

- `if strings.Contains(err.Error(), "not found")` / regex-on-message — replace with a sentinel + `errors.Is` or a typed error + `errors.As`.
- Returning errors that callers need to distinguish, but with no sentinel and no type — callers will be forced into string matching.
- Custom error type whose only difference from `errors.New` is the formatted message (no extra fields, no behaviour) — collapse to a sentinel.
- Multiple sentinel constants for the same condition (`ErrNotFound1`, `ErrNotFound2`) — collapse.

---

## #adding-information-to-errors — Adding information to errors

> When adding information to errors, avoid redundant information that the underlying error already provides.

### Good — annotation adds context, not duplication

```go
if err := os.Open("settings.txt"); err != nil {
    return fmt.Errorf("launch codes unavailable: %v", err)
}
// Output: launch codes unavailable: open settings.txt: no such file or directory
```

> Here, "launch codes unavailable" adds specific meaning to the `os.Open` error that's relevant to the current function's context, without duplicating the underlying file path information.

### Bad — annotation duplicates the underlying error

```go
if err := os.Open("settings.txt"); err != nil {
    return fmt.Errorf("could not open settings.txt: %v", err)
}
// Output: could not open settings.txt: open settings.txt: no such file or directory
```

(`open settings.txt` appears twice.)

### Bad — content-free annotation

> Don't add an annotation if its sole purpose is to indicate a failure without adding new information.

```go
return fmt.Errorf("failed: %v", err) // just return err instead
```

### Smells

- `fmt.Errorf("error: %v", err)` / `fmt.Errorf("failed: %v", err)` / `fmt.Errorf("oops: %v", err)` — content-free; return `err` directly.
- Annotation that re-states the file path / URL / argument the underlying error already includes (`fmt.Errorf("could not load %s: %v", path, err)` when the underlying error already mentions `path`).
- Annotation that re-states the function name (`fmt.Errorf("Parse: %v", err)` from inside `Parse()`) — caller already has stack context.
- `errors.Wrap(err, "failed")` (deprecated `pkg/errors` style) — modernize to `fmt.Errorf("...: %w", err)`.

---

## #placement-of-w-in-errors — Placement of `%w` in errors

> Prefer to place `%w` at the end of an error string *if* you are to use error wrapping with the `%w` formatting verb.

### Good — `%w` at the end mirrors the chain

```go
err1 := fmt.Errorf("err1")
err2 := fmt.Errorf("err2: %w", err1)
err3 := fmt.Errorf("err3: %w", err2)
fmt.Println(err3) // err3: err2: err1
```

> Therefore, in order for error text to mirror error chain structure, prefer placing the `%w` verb at the end with the form `[...]: %w`.

### #sentinel-error-placement — Sentinel error placement

> An exception to this rule is when wrapping sentinel errors. A sentinel error is an error that serves as a primary categorization of a failure.

> Placing the `%w` verb at the beginning of the error string can improve readability by immediately identifying the category of the error.

#### Good — sentinel at the start

```go
package parser

var ErrParse = fmt.Errorf("parse error")

var ErrParseInvalidHeader = fmt.Errorf("%w: invalid header", ErrParse)

func parseHeader() error {
    err := checkHeader()
    return fmt.Errorf("%w: invalid character in header: %v", ErrParseInvalidHeader, err)
}

err := fmt.Errorf("%w: couldn't find fortune database: %v", ErrInternal, err)
```

#### Bad — sentinel buried at the end

```go
package parser

var ErrParse = fmt.Errorf("parse error")

var ErrParseInvalidHeader = fmt.Errorf("%w: invalid header", ErrParse)

func parseHeader() error {
    err := checkHeader()
    return fmt.Errorf("invalid character in header: %v: %w", err, ErrParseInvalidHeader)
}

var ErrInternal = status.Error(codes.Internal, "internal")
err2 := fmt.Errorf("couldn't find fortune database: %v: %w", err, ErrInternal)
```

> When you place it at the end, it makes it harder to identify the error category when reading the error text, as it's buried in the specific error details.

### Decision tree for `%w` placement

| Wrapping... | Place `%w`... | Form |
|---|---|---|
| A non-sentinel underlying error | At the end | `fmt.Errorf("doing thing X: %w", err)` |
| A sentinel error (categorization) | At the start | `fmt.Errorf("%w: doing thing X failed: %v", ErrSentinel, err)` |
| Both a sentinel and a wrapped error | Sentinel at start (with `%w`); underlying with `%v` | as above |

### Smells

- Wrap form `fmt.Errorf("%w: %v", err, sentinel)` — sentinel at the *end*; flip per the above.
- Multiple `%w` in one `fmt.Errorf` — Go ≥ 1.20 supports it, but each `%w` adds an `errors.Is` target; verify intent.
- `fmt.Errorf("%v", err)` where `%w` was intended — drops the wrap chain; callers can't `errors.Is`.
- Sentinel defined as `errors.New("...")` but always wrapped without `%w` — sentinel is unreachable via `errors.Is`.

---

## #logging-errors — Logging errors

> Functions sometimes need to tell an external system about an error without propagating it to their callers.

Key rules:

- > Like good test failure messages, log messages should clearly express what went wrong and help the maintainer by including relevant information to diagnose the problem.
- > Avoid duplication. If you return an error, it's usually better not to log it yourself but rather let the caller handle it.
- > Be careful with PII. Many log sinks are not appropriate destinations for sensitive end-user information.
- > Use `log.Error` sparingly. ERROR level logging causes a flush and is more expensive than lower logging levels.

### #custom-verbosity-levels — Custom verbosity levels

> Use verbose logging (`log.V`) to your advantage. Verbose logging can be useful for development and tracing.

Recommendation: minimal info at `V(1)`, more tracing at `V(2)`, large internal state dumps at `V(3)`.

#### Good — guard expensive `Explain()` so it doesn't run when log is suppressed

```go
for _, sql := range queries {
    log.V(1).Infof("Handling %v", sql)
    if log.V(2) {
        log.Infof("Handling %v", sql.Explain())
    }
    sql.Run(...)
}
```

#### Bad — `Explain()` runs even when `V(2)` is off

```go
log.V(2).Infof("Handling %v", sql.Explain())
// sql.Explain called even when this log is not printed.
```

The argument is *evaluated* before being passed; only the *output* is suppressed.

### Smells

- Function returns an error AND logs the same error — drop the log; let the caller decide.
- `log.Errorf` for routine, recoverable conditions (4xx HTTP responses, expected validation failures) — use a lower level.
- `log.V(N).Infof("%v", expensiveCall())` where `expensiveCall` always runs — wrap with `if log.V(N) { ... }` (or `if log.V(N).Enabled() {}`) when the cost is real.
- PII (emails, names, IDs, raw payloads) logged where the sink isn't authorized for that data class.
- `log.Print(err)` (no level, no message context) — at minimum, `log.Errorf("doing X: %v", err)`.

---

## #program-initialization — Program initialization

> Program initialization errors (such as bad flags and configuration) should be propagated upward to `main`, which should call `log.Exit` with an error that explains how to fix the error.

> In these cases, `log.Fatal` should not generally be used, because a stack trace that points at the check is not likely to be as useful as a human-generated, actionable message.

So at startup:

| Failure condition | Use |
|---|---|
| Bad flag value, missing required env, malformed config file | `log.Exit("...")` (no stack — the stack would not help) |
| Internal invariant broken before `main` returns (assertion failure, "this can't happen" condition) | `log.Fatal("...")` (with stack) |

The principle: at startup, the *user* needs to know how to fix their input — not where in the binary the validation lives.

### Smells

- `log.Fatalf("invalid -port: %v", err)` for a flag-parse error in `main` — should be `log.Exit` (or just print to stderr + `os.Exit(1)`).
- `panic("config file not found")` at startup — `log.Exit` instead.
- A library function called at startup that calls `log.Exit` directly — let `main` decide.
- Startup error messages without remediation guidance (`"invalid config"` instead of `"-config: file 'app.toml' not found; create one or pass --config=PATH"`).

---

## #program-checks-and-panics — Program checks and panics

> As stated in the decision against panics, standard error handling should be structured around error return values.

> Libraries should prefer returning an error to the caller rather than aborting the program, especially for transient errors.

When *can* you terminate the program?

> It is occasionally necessary to perform consistency checks on an invariant and terminate the program if it is violated.

> In general, this is only done when a failure of the invariant check means that the internal state has become unrecoverable.

> The most reliable way to do this in the Google codebase is to call `log.Fatal`.

Why not `panic`?

> Using `panic` in these cases is not reliable, because it is possible for deferred functions to deadlock or further corrupt internal or external state.

> Similarly, resist the temptation to recover panics to avoid crashes, as doing so can result in propagating a corrupted state.

### Decision tree

| Situation | Use |
|---|---|
| Recoverable error (most cases) | Return `error` |
| Library code, transient failure | Return `error` |
| Invariant violated, state unrecoverable | `log.Fatal` |
| "I want to recover panics so my server stays up" | **Don't** — corrupted state will leak |

### Smells

- `panic(fmt.Errorf(...))` for invariant checks — use `log.Fatal` instead.
- `defer recover()` wrapping unrelated business logic to "prevent crashes" — the recovered state may be corrupt; let it crash and let supervisor restart.
- Library function panicking on bad input — return `error` so the caller decides.
- `runtime.Goexit()` used as a panic alternative — same problem; use `log.Fatal` if termination is truly needed.

---

## #when-to-panic — When to panic

> The standard library panics on API misuse. For example, `reflect` issues a panic in many cases where a value is accessed in a way that suggests it was misinterpreted.

> Code review and tests should discover such bugs, which are not expected to appear in production code.

> Another case in which panics can be useful, though uncommon, is as an internal implementation detail of a package which always has a matching recover in the callchain.

> The key attribute of this design is that these **panics are never allowed to escape across package boundaries** and do not form part of the package's API.

### Decision tree

| Panic use case | OK? |
|---|---|
| API misuse the caller could not recover from (`reflect.ValueOf(nil).Type()`-style) | OK — but rare; prefer returning an error |
| Internal control flow within a package, with matching `recover` in the same package | OK (uncommon) — but never expose as a public contract |
| Normal error path | **Never** |
| Expected runtime failure (network, disk, parse) | **Never** |
| "Impossible" condition you want to terminate on | Use `log.Fatal`, not `panic` (per `#program-checks-and-panics`) |

### Smells

- `panic` in any non-test code path that handles user input or external I/O.
- Public API contract that "panics on bad input" — should return an error.
- Panic-recover pair across package boundaries (panic in `pkg/foo`, recover in `cmd/main`) — fragile; refactor into errors.
- A library that panics with no recover at all in its design — caller's process dies for what should be a regular error.

---

## Consolidated review checklist (7 sub-rules)

### Error structure & wrapping

- [ ] Errors that callers must distinguish are exposed as sentinel constants or typed errors — not by string matching.
- [ ] No `strings.Contains(err.Error(), ...)` or regex-on-`err.Error()` in caller code.
- [ ] `errors.Is` used for sentinel comparison; `errors.As` for typed-error inspection.
- [ ] Wrap annotations add context, not duplication.
- [ ] No content-free wraps (`fmt.Errorf("failed: %v", err)`).
- [ ] `%w` placed at the **end** when wrapping a non-sentinel underlying error (`"doing X: %w"`).
- [ ] `%w` placed at the **start** when wrapping a sentinel (`"%w: doing X failed: %v"`) so the category is immediately readable.

### Logging errors

- [ ] Returned errors are not also logged at the same call site (caller decides).
- [ ] `log.Error` reserved for genuinely-error conditions, not routine recoverables.
- [ ] Expensive arguments to `log.V(N).Infof(...)` are guarded with `if log.V(N) { ... }` to avoid evaluation when suppressed.
- [ ] No PII logged where the sink isn't authorized for it.

### Initialization & panics

- [ ] Startup config / flag errors use `log.Exit` with a remediation-friendly message (not `log.Fatal`, not `panic`).
- [ ] `log.Fatal` reserved for invariant violations where state is unrecoverable.
- [ ] No `panic` for normal error handling, transient failures, or library errors.
- [ ] No defensive `recover()` blocks intended to "prevent crashes" — corrupt state should propagate.
- [ ] Internal `panic`/`recover` patterns (rare) never escape package boundaries.

## Suggested finding phrasing (advisory)

- "`if strings.Contains(err.Error(), \"not found\") { ... }` — Best Practices §error-structure: define a sentinel (`ErrNotFound`) or typed error and use `errors.Is` / `errors.As`."
- "`fmt.Errorf(\"could not open settings.txt: %v\", err)` after `os.Open(\"settings.txt\")` — Best Practices §adding-information-to-errors: the underlying error already names the file; either drop the duplication (`fmt.Errorf(\"launch codes unavailable: %v\", err)`) or return `err` directly."
- "`fmt.Errorf(\"failed: %v\", err)` — Best Practices: content-free annotation; `return err` instead."
- "`fmt.Errorf(\"%w: \", ErrSomething)` is at the start, but the wrapped value isn't a sentinel — flip to `fmt.Errorf(\"doing X: %w\", underlying)`. (Best Practices §placement-of-w-in-errors)."
- "Sentinel `ErrInternal` wrapped at the end (`\"...: %w\"`) — Best Practices §sentinel-error-placement: put the sentinel at the start (`\"%w: ...\"`) so the category reads first."
- "Function returns `err` and also calls `log.Errorf(\"%v\", err)` — Best Practices §logging-errors: drop the log; let the caller handle it."
- "`log.V(2).Infof(\"%v\", sql.Explain())` — `Explain()` runs unconditionally; wrap with `if log.V(2) { log.Infof(\"%v\", sql.Explain()) }`."
- "`log.Fatalf(\"invalid -port: %v\", err)` in `main` — Best Practices §program-initialization: use `log.Exit` with a fix-it message; the stack trace doesn't help the user."
- "`panic(err)` in non-test, non-init code — Best Practices §when-to-panic: return `error` instead; reserve `log.Fatal` for unrecoverable invariant violations."
- "`defer func() { if r := recover(); r != nil { log.Printf(\"recovered: %v\", r) } }()` wrapping business logic — Best Practices §program-checks-and-panics: corrupted state may persist; let the process die and the supervisor restart it."

## Sources

- <https://google.github.io/styleguide/go/best-practices#error-handling> — Google Go Best Practices, Error handling section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#errors> — Style Decisions, Errors (foundational rules — sibling skill `decisions-review:04-errors.md`)
- <https://google.github.io/styleguide/go/decisions#dont-panic> — Decisions §dont-panic (covered in `decisions-review:05-language.md#dont-panic`)
- <https://pkg.go.dev/errors> — Go `errors` package (`errors.Is`, `errors.As`, `%w`)
- <https://pkg.go.dev/fmt#Errorf> — `fmt.Errorf` (`%w` semantics)
