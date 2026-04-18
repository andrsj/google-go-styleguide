---
section: "Errors"
number: 4
category: actionable
rules:
  - returning-errors
  - error-strings
  - handle-errors
  - in-band-errors
  - indent-error-flow
sources:
  - https://google.github.io/styleguide/go/decisions#errors
---

# Section 04 — Errors

## What this section covers

Five concrete decisions on error handling: how to return errors, how to write error message strings, that errors must be handled (not discarded), the prohibition on in-band sentinel values (`-1`, `""`, `nil`), and the early-return / no-`else` indentation pattern.

> [!IMPORTANT]
> This section pairs with the Style Guide's `#01 Clarity` (the `if err != nil { return err }` pattern is the canonical clarity tool) and the broader Google Best Practices on error wrapping (covered in sibling skill `best-practices-review`).

## #returning-errors — Returning errors

> Use `error` to signal that a function can fail. By convention, `error` is the last result parameter.

```go
// Good:
func Good() error { /* ... */ }
```

> Returning a `nil` error is the idiomatic way to signal a successful operation that could otherwise fail.

```go
// Good:
func GoodLookup() (*Result, error) {
    // ...
    if err != nil {
        return nil, err
    }
    return res, nil
}
```

> Exported functions that return errors should return them using the `error` type. Concrete error types are susceptible to subtle bugs: a concrete `nil` pointer can get wrapped into an interface and thus become a non-nil value.

```go
// Bad:
func Bad() *os.PathError { /*...*/ }
```

The "concrete `nil` pointer becomes non-nil interface" trap: if `Bad()` returns `(*os.PathError)(nil)` and the caller assigns to an `error` variable, `err != nil` is true even though the pointer is `nil`. Always return `error`, not the concrete type.

### Smells

- Exported function returns `*MyError` (or any concrete error type) — change to `error`.
- `error` is not the last return value (`func Foo() (error, *Result)`).
- Function with a body that does `return MyError{}` directly when the caller expects `error` — fine when not exported, but risky for the same nil-interface reason.
- Functions that don't return `error` but signal failure through panics, sentinel return values, or globals — see `#in-band-errors` and `#dont-panic` (in section 05).

---

## #error-strings — Error strings

> Error strings should not be capitalized (unless beginning with an exported name, a proper noun or an acronym) and should not end with punctuation.

```go
// Bad:
err := fmt.Errorf("Something bad happened.")

// Good:
err := fmt.Errorf("something bad happened")
```

The reason: error strings get composed into longer messages (`"failed to fetch user 42: " + err.Error()`). Lowercased, no-trailing-period strings concatenate cleanly.

> The style for the full displayed message (logging, test failure, API response, or other UI) depends, but should typically be capitalized.

```go
// Good:
log.Infof("Operation aborted: %v", err)
log.Errorf("Operation aborted: %v", err)
t.Errorf("Op(%q) failed unexpectedly; err=%v", args, err)
```

So: **error strings** are lowercase + no period; **log/UI/test messages** that *contain* an error are full sentences (capitalized, no period at the end).

### Smells

- `fmt.Errorf("Something bad")` — uppercase first letter (and not a proper noun / acronym / exported name).
- `fmt.Errorf("something bad happened.")` — trailing period.
- `errors.New("Failed to parse JSON.")` — both smells.
- Error string starts with a function/package name to provide context (`fmt.Errorf("ParseURL: invalid scheme")`) — fine in some Google codebases (`pkg: action: detail` convention), but the leading word is the package noun, not a sentence-starting capital.
- Error string includes ANSI colors, emoji, or formatting — leave styling to the logger/UI.

### Allowed exceptions to the lowercase rule

Capitalize when the first word is:
- An **exported name** (`fmt.Errorf("Open failed: %v", err)` if `Open` is the function being referenced)
- A **proper noun** (`fmt.Errorf("HTTP request failed")` — `HTTP` is an acronym, see #initialisms)
- An **acronym** (`fmt.Errorf("URL parse failed")`, `fmt.Errorf("ID not found")`)

---

## #handle-errors — Handle errors

> Code that encounters an error should make a deliberate choice about how to handle it. It is not usually appropriate to discard errors using `_` variables.

> In the rare circumstance where it is appropriate to ignore or discard an error (for example a call to `(*bytes.Buffer).Write` that is documented to never fail), an accompanying comment should explain why this is safe.

```go
// Good:
var b *bytes.Buffer

n, _ := b.Write(p) // never returns a non-nil error
```

The standard for ignoring an error is: (a) the documented contract guarantees no error, AND (b) you put a comment explaining why.

### Smells

- `result, _ := someFunc()` with no comment — discarding without justification.
- `defer file.Close()` (where `Close()` returns an error) — a common pattern, but the error is silently ignored. For files being **read**, it's usually acceptable; for files being **written**, the error matters and should be checked. Decisions doesn't ban `defer f.Close()`, but flag write-side cases.
- Catch-and-log without recovery: `if err != nil { log.Printf("err: %v", err) }` and continuing as if nothing happened. That's "handling" only in the loosest sense — verify the next code path can actually proceed safely.
- `if err == nil { … } else { return nil }` — flip the condition (see `#indent-error-flow`).
- Custom error wrapping that loses the underlying error (no `%w`, no `errors.Is/As` access). Decisions in this section doesn't mandate `%w`, but Best Practices does — flag for review under the sibling skill.

---

## #in-band-errors — In-band errors

> In C and similar languages, it is common for functions to return values like `-1`, `null`, or the empty string to signal errors or missing results. This is known as in-band error handling.

```go
// Bad:
// Lookup returns the value for key or -1 if there is no mapping for key.
func Lookup(key string) int
```

> Failing to check for an in-band error value can lead to bugs and can attribute errors to the wrong function.

```go
// Bad:
// The following line returns an error that Parse failed for the input value,
// whereas the failure was that there is no mapping for missingKey.
return Parse(Lookup(missingKey))
```

> Go's support for multiple return values provides a better solution (see the Effective Go section on multiple returns). Instead of requiring clients to check for an in-band error value, a function should return an additional value to indicate whether its other return values are valid. This return value may be an error or a boolean when no explanation is needed, and should be the final return value.

```go
// Good:
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```

> This API prevents the caller from incorrectly writing `Parse(Lookup(key))` which causes a compile-time error, since `Lookup(key)` has 2 outputs.

```go
// Good:
value, ok := Lookup(key)
if !ok {
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```

> Some standard library functions, like those in package `strings`, return in-band error values. This greatly simplifies string-manipulation code at the cost of requiring more diligence from the programmer. In general, Go code in the Google codebase should return additional values for errors.

### Smells

- Function returns `int` and "uses `-1` for not-found" or `0` for "no value" — change to `(int, bool)` or `(int, error)`.
- Function returns `string` and uses `""` to signal absence — same fix.
- Function returns `*Foo` and uses `nil` for "not found" without an `error` — fine in some idioms (e.g., `cache.Get`), but the doc prefers `(value, ok)` or `(value, error)`. Borderline; flag if the absence-vs-error distinction matters.
- A magic sentinel value (`-1`, `MaxInt`, `time.Time{}`) being used as "no result." The `time.IsZero()` idiom is acceptable for `time.Time`; raw integer sentinels generally are not.
- Code that calls `Parse(Lookup(key))` style chaining with no `ok`/`err` check between calls — exactly the bug-attribution problem the doc warns about.

---

## #indent-error-flow — Indent error flow

> Handle errors before proceeding with the rest of your code. This improves the readability of the code by enabling the reader to find the normal path quickly. This same logic applies to any block which tests a condition then ends in a terminal condition (e.g., `return`, `panic`, `log.Fatal`).

> Code that runs if the terminal condition is not met should appear after the `if` block, and should not be indented in an `else` clause.

```go
// Good:
if err != nil {
    // error handling
    return // or continue, etc.
}
// normal code
```

```go
// Bad:
if err != nil {
    // error handling
} else {
    // normal code that looks abnormal due to indentation
}
```

The same shape applies to any terminal condition — `panic`, `log.Fatal`, `os.Exit`, `continue`, `break`. The "happy path" should always be at the lowest indentation in the function.

### Smells

- `if err != nil { … } else { … }` — invert: `if err != nil { return … }` then unindent the rest.
- Nested error checks creating a "pyramid of doom" — each `if err == nil` should be flipped to `if err != nil { return err }`.
- Validation chains like `if x != "" { if y != "" { if z != "" { … } } }` — same fix; early return on each negative case.
- Mid-function `if cond { … } else { … }` where the `else` is the dominant path. Either rename the boolean (`isAnomaly` → `isNormal`) so the `if` body is the rare path, or invert.
- Long `else` blocks (10+ lines) — almost always should be unindented after an early return on the inverse.

---

## Consolidated review checklist (all 5 rules)

- [ ] Functions that can fail return `error` as the last return value.
- [ ] Exported functions return the `error` interface, not a concrete `*MyError` type.
- [ ] Error strings are lowercase and have no trailing period (unless starting with a proper noun, acronym, or exported name).
- [ ] Log/UI/test messages that contain an error are full capitalized sentences.
- [ ] No `_, _ := someFunc()` discarding without an explanatory comment proving it's safe.
- [ ] No in-band sentinel values (`-1`, `""`, `nil` without an `error`) used as "no result" — use `(value, ok)` or `(value, error)` instead.
- [ ] Error handling uses early returns; no `if err != nil { … } else { … }` shape.
- [ ] Happy path is at the lowest indentation in the function.
- [ ] No "pyramid of doom" — each negative condition is flipped to an early return.

## Suggested finding phrasing

- "`func Open() *os.PathError` — Decisions §returning-errors: exported function returns concrete error type; change to `error` to avoid the nil-interface trap."
- "`fmt.Errorf(\"Failed to parse.\")` — Decisions §error-strings: lowercase first word, no trailing period (`\"failed to parse\"`)."
- "`_, _ = io.Copy(dst, src)` with no comment — Decisions §handle-errors: discarding error needs a justifying comment, or actually handle the error."
- "`func Lookup(k string) int` returns `-1` for missing — Decisions §in-band-errors: change to `(int, bool)` or `(int, error)`."
- "`if err != nil { … } else { /* normal path */ }` — Decisions §indent-error-flow: early-return the error and unindent the normal path."
- "Pyramid of nested `if err == nil` checks — Decisions §indent-error-flow: invert each, return early on the error case."
- "`Parse(Lookup(key))` chained with no separate `ok`/`err` check — Decisions §in-band-errors: use `value, ok := Lookup(key)` and check `ok` before calling `Parse`."

## Sources

- <https://google.github.io/styleguide/go/decisions#errors> — Google Go Style Decisions, Errors section (the canonical text quoted above)
- <https://go.dev/doc/effective_go#multiple-returns> — Effective Go, multiple returns (referenced by `#in-band-errors`)
- <https://pkg.go.dev/errors> — Go `errors` package (`errors.Is`, `errors.As`, `%w` wrapping; covered in detail by sibling skill `best-practices-review`)
