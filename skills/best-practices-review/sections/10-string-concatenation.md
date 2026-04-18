---
section: "String concatenation"
number: 10
category: actionable
rules:
  - prefer-plus-for-simple-cases
  - prefer-fmt-sprintf-when-formatting
  - prefer-strings-builder-for-piecemeal
  - constant-strings
sources:
  - https://google.github.io/styleguide/go/best-practices#string-concatenation
---

# Section 10 — String concatenation (Best Practices)

## What this section covers

Four recommendations for *which* string-building tool to reach for: `+` for trivial joins, `fmt.Sprintf` for formatted output, `strings.Builder` for incremental construction in loops, and backtick literals for multi-line constants.

> [!IMPORTANT]
> The performance argument matters: `+` and `fmt.Sprintf` are **quadratic** when called repeatedly to grow a string (each call allocates a new buffer and copies). `strings.Builder` is amortized **linear**. Get this wrong in a hot loop and you'll see it on a profile.

## Overview

> There are several ways to concatenate strings in Go. Some examples include:
>
> - The "+" operator
> - `fmt.Sprintf`
> - `strings.Builder`
> - `text/template`
> - `safehtml/template`

> Though there is no one-size-fits-all rule for which to choose, the following guidance outlines when each method is preferred.

## #prefer-plus-for-simple-cases — Prefer `+` for simple cases

> Prefer using "+" when concatenating few strings. This method is syntactically the simplest and requires no import.

```go
// Good:
key := "projectid: " + p
```

### Smells

- `fmt.Sprintf("%s%s", a, b)` for a 2-string join with no formatting — `a + b`.
- `strings.Builder` ceremony for a single-shot 3-piece join — `+` is fine.
- `strings.Join([]string{a, b}, "")` — overkill for two pieces.

---

## #prefer-fmt-sprintf-when-formatting — Prefer `fmt.Sprintf` when formatting

> Prefer using `fmt.Sprintf` when building a complex string with formatting. Using many "+" operators may obscure the end result.

### Good

```go
str := fmt.Sprintf("%s [%s:%d]-> %s", src, qos, mtu, dst)
```

### Bad

```go
bad := src.String() + " [" + qos.String() + ":" + strconv.Itoa(mtu) + "]-> " + dst.String()
```

The `+`-and-`strconv` chain hides the template — the reader has to mentally re-assemble the literal punctuation between the values to picture the output. `fmt.Sprintf` keeps the shape visible.

### Best practice for `io.Writer` destinations

> When the output of the string-building operation is an `io.Writer`, don't construct a temporary string with `fmt.Sprintf` just to send it to the Writer. Instead, use `fmt.Fprintf` to emit to the Writer directly.

```go
// Bad:
fmt.Fprint(w, fmt.Sprintf("hello %s", name))

// Good:
fmt.Fprintf(w, "hello %s", name)
```

### When formatting is more complex

> When the formatting is even more complex, prefer [`text/template`](https://pkg.go.dev/text/template) or [`safehtml/template`](https://pkg.go.dev/github.com/google/safehtml/template) as appropriate.

The boundary: as soon as you're branching, looping, or escaping inside the template logic, move to `text/template` (HTML — `safehtml/template`).

### Smells

- Long `+` chains gluing literals + values + literals + values — switch to `fmt.Sprintf`.
- `strconv.Itoa(...)` inside a `+` chain for formatting purposes — use `%d` in a Sprintf instead.
- `w.Write([]byte(fmt.Sprintf(...)))` — use `fmt.Fprintf(w, ...)`.
- Hand-rolled HTML escaping (`strings.Replace` for `<`, `>`, `&`) — use `safehtml/template` (or at least `html/template`).

---

## #prefer-strings-builder-for-piecemeal — Prefer `strings.Builder` for constructing a string piecemeal

> Prefer using `strings.Builder` when building a string bit-by-bit. `strings.Builder` takes amortized linear time, whereas "+" and `fmt.Sprintf` take quadratic time when called sequentially to form a larger string.

### Good

```go
b := new(strings.Builder)
for i, d := range digitsOfPi {
    fmt.Fprintf(b, "the %d digit of pi is: %d\n", i, d)
}
str := b.String()
```

`strings.Builder` implements `io.Writer`, so `fmt.Fprintf(b, ...)` works — no need to allocate intermediate strings per iteration.

### Why quadratic?

`s = s + "x"` allocates a fresh underlying buffer of length `len(s)+1` and copies the old `s` into it. Do that N times and you've copied roughly `N²/2` bytes total. `strings.Builder` keeps a growing buffer (capacity doubles), so N appends cost `O(N)` amortized.

### Smells

- `for ... { s = s + ... }` building a result string — switch to `strings.Builder`.
- `for ... { s += ... }` — same anti-pattern.
- `for ... { s = fmt.Sprintf("%s%s", s, x) }` — same again, dressed up.
- `strings.Builder` reset between uses without calling `b.Reset()` — leaves stale data.
- `strings.Builder` declared but only one `WriteString` call before `String()` — overkill; just use `+` or `fmt.Sprintf`.

### Performance hint

If you know roughly how many bytes you'll append, call `b.Grow(n)` once up-front:

```go
var b strings.Builder
b.Grow(8 * len(items))  // estimate
for _, x := range items {
    b.WriteString(x)
}
return b.String()
```

This eliminates the doubling-step copies. Only worth it when the count is large *and* roughly known.

---

## #constant-strings — Constant strings

> Prefer to use backticks (`` ` ``) when constructing constant, multi-line string literals.

### Good

```go
usage := `Usage:

custom_tool [args]`
```

### Bad

```go
usage := "" +
    "Usage:\n" +
    "\n" +
    "custom_tool [args]"
```

The backtick form preserves whitespace literally, doesn't need explicit `\n`, doesn't need `+` chaining, and makes the multi-line shape visible at a glance.

### Smells

- Multi-line string built via `\n` + `+` joins — switch to a backtick literal.
- Multi-line backtick literal *inside* a function indented deeply (each line picks up the leading indentation) — fine, but be aware the indentation is part of the string. If you need to dedent, use `strings.TrimSpace` + literal trimming, or move the constant to package scope.
- Multi-line `+` chains for documentation, help text, embedded SQL/JSON/YAML — switch to backticks.
- Backtick literal that wants to embed a backtick — backticks don't allow escaping; concatenate with `+ "`" +` or use a regular string with `\``.

### When NOT to use backticks

- The string itself contains a backtick character — must use the regular `"..."` form (or split + concatenate).
- Indentation matters and you can't move the constant to package scope (the leading whitespace from the source file's indentation gets baked in).

---

## Decision tree (what to use when)

| Situation | Use |
|---|---|
| 2-3 pieces, no formatting | `+` |
| Format string with values | `fmt.Sprintf(format, args...)` |
| Format string + writing to an `io.Writer` | `fmt.Fprintf(w, format, args...)` (skip the temp string) |
| Loop building a result string | `strings.Builder` |
| Complex template with branches/loops | `text/template` (HTML: `safehtml/template`) |
| Multi-line constant (help text, SQL, etc.) | Backtick literal `` `...` `` |
| Multi-line constant containing a backtick | Regular `"..."` with `\n`s, or `+`-joined |

---

## Consolidated review checklist (4 sub-rules)

- [ ] Two-piece string joins use `+`, not `fmt.Sprintf("%s%s", a, b)`.
- [ ] Formatted output uses `fmt.Sprintf` (or `fmt.Fprintf` when the destination is an `io.Writer`).
- [ ] No long `+` chains with type-conversion functions (`strconv.Itoa`) — switch to `fmt.Sprintf` with `%d`.
- [ ] Loops building a string use `strings.Builder` (or `bytes.Buffer` for `[]byte`), not `+=` / `+`.
- [ ] `b.Grow(n)` used in `strings.Builder` when the final size is roughly known and the loop is hot.
- [ ] Multi-line constants use backtick literals, not `\n` + `+` chains.

## Suggested finding phrasing (advisory)

- "`fmt.Sprintf(\"%s%s\", a, b)` for two-piece concatenation — Best Practices §prefer-plus-for-simple-cases: `a + b` is simpler and faster."
- "`src.String() + \" [\" + qos.String() + \":\" + strconv.Itoa(mtu) + \"]\"...` long `+` chain — Best Practices §prefer-fmt-sprintf-when-formatting: switch to `fmt.Sprintf(\"%s [%s:%d]\", src, qos, mtu)` to make the template visible."
- "`w.Write([]byte(fmt.Sprintf(...)))` allocates a temp string — Best Practices: use `fmt.Fprintf(w, ...)` directly."
- "`for ... { result = result + line }` — Best Practices §prefer-strings-builder-for-piecemeal: this is O(N²); use `strings.Builder` for amortized linear time."
- "Multi-line `usage := \"Usage:\\n\" + \"\\n\" + \"custom_tool [args]\"` — Best Practices §constant-strings: use a backtick literal for multi-line constants."
- "Hand-rolled HTML escaping with `strings.Replace` — Best Practices: use `safehtml/template` (or at least `html/template`) for safe escaping."

## Sources

- <https://google.github.io/styleguide/go/best-practices#string-concatenation> — Google Go Best Practices, String concatenation section (the canonical text quoted above)
- <https://pkg.go.dev/strings#Builder> — `strings.Builder`
- <https://pkg.go.dev/fmt#Sprintf> — `fmt.Sprintf` / `fmt.Fprintf`
- <https://pkg.go.dev/text/template> — `text/template` (referenced for complex formatting)
- <https://pkg.go.dev/github.com/google/safehtml/template> — `safehtml/template` (referenced for HTML — safer than `html/template` for XSS)
- <https://google.github.io/styleguide/go/index.html#gotip> — *GoTip #29: Building Strings Efficiently* (referenced)
