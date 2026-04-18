---
guideline: "Line length"
number: 8
category: actionable
sources:
  - https://google.github.io/styleguide/go/guide
---

# #08 — Line length

## What it requires

The Style Guide does **not** set a hard column limit. The rule is shaped by intent, not characters:

> There is no fixed line length for Go source code.

> If a line feels too long, prefer refactoring instead of splitting it.

> If it is already as short as it is practical for it to be, the line should be allowed to remain long.

And two specific anti-patterns to avoid:

> Do not split a line:
> - Before an indentation change (e.g., function declaration, conditional)
> - To make a long string (e.g., a URL) fit into multiple shorter lines

In short: **refactor long lines, don't wrap them**. If refactoring isn't possible, leave the line long.

## What to look for (code smells)

- A long line that has been mechanically wrapped at ~80/100/120 columns mid-expression — the Guide's preferred move is to extract a variable, not to wrap.
- A long URL string broken into concatenated parts (`"https://example.com/" +` `"api/v1/" +` `"resource"`) just to fit a width — the Guide explicitly forbids this; keep the URL on one line.
- A function signature wrapped *before* the opening brace's indent change:
  ```go
  func DoSomething(
      a int, b int,
  ) error {
  ```
  The Guide says "Do not split a line before an indentation change." Either keep the signature on one line, or refactor (fewer parameters, a config struct, etc.).
- A long `if` condition wrapped *before* the brace:
  ```go
  if a == 1 &&
      b == 2 {
  ```
  Same anti-pattern. Refactor by extracting a `cond := …` variable, or reshape the condition.
- Chained method calls split mid-chain to fit width (`x.Foo(). Bar(). Baz()` across lines) — extract intermediate variables instead.
- `gofmt` already broke the line for you (e.g., long composite literal field-per-line, struct tag across lines) — leave it alone, that's the formatter's call.
- Soft column rulers in editor configs that auto-wrap — fine for visual reference, not a rule to enforce.

## Good patterns

### Refactor, don't wrap

Instead of:

```go
return fmt.Errorf("failed to fetch user %d from %s with token %s: %w", id, host, tok, err)
```

if it feels too long, extract:

```go
fields := []any{id, host, tok, err}
return fmt.Errorf("failed to fetch user %d from %s with token %s: %w", fields...)
```

or split into a helper that names the operation. Don't wrap the original literal across lines just for width.

### Long string stays one line

```go
const docsURL = "https://google.github.io/styleguide/go/guide#line-length"
```

Not:

```go
const docsURL = "https://google.github.io/styleguide/go/" +
    "guide#line-length"
```

### Function signature: refactor or keep on one line

If a signature is too long, the typical Go answer is *fewer parameters* — pass an `Options` struct, group related args, or split the function:

```go
type FetchOptions struct {
    Host  string
    Token string
    Retry int
}

func Fetch(ctx context.Context, id int64, opts FetchOptions) (*User, error) { … }
```

…rather than:

```go
func Fetch(ctx context.Context, id int64,
    host string, token string, retry int) (*User, error) {
```

### Long condition: extract

```go
isStale := time.Since(req.Created) > maxAge && req.RetryCount < maxRetries
if isStale {
    …
}
```

beats wrapping the original `&&`-chain mid-expression.

## Review checklist

- [ ] Are wrapped lines actually unavoidable, or could a `:=` extraction shorten the original?
- [ ] Are URL / file-path / SQL-fragment string literals kept on a single line?
- [ ] Are function signatures kept on one line, or — if too long — refactored to take fewer parameters (or a struct)?
- [ ] Are `if` / `for` / `switch` conditions kept on one line, or — if too long — refactored into a named boolean?
- [ ] Are method chains either short enough to fit, or split via intermediate `:=` variables?
- [ ] Is the wrapped form one that `gofmt` produced (e.g., field-per-line composite literal)? If yes, leave it.

## Tooling notes

Parts of #08 are mechanically detectable; parts require human judgment. Be aware of the **anti-tooling trap** — most off-the-shelf line-length linters contradict the rule.

### Tools that fit (use these)

- **`gofmt`** — already rejoins some wrapped expressions and enforces the canonical break points. Required by #06 anyway.
- **`gofumpt`** — stricter superset; tightens single-statement composites and removes redundant wrapping that `gofmt` leaves alone. See #06's note on `gofumpt`.
- **`goimports`** — handles import grouping; tangentially related (line layout consistency).
- **A custom AST check** for the two specific anti-patterns the Guide names:
  - String literals concatenated with `+` across adjacent lines that produce a single URL/path/SQL string (`"https://..." + "/api/..." + "?x=1"`).
  - Function signatures wrapped before the opening brace's indentation change (when not produced by `gofmt`).
  These are genuinely linter-shaped and would be a valuable rule, but no widely-used golangci-lint check targets them today.

### Tools that **don't** fit (caveat)

- **`lll`** (golangci-lint) — enforces a max line length. **Directly contradicts** the Guide's "no fixed line length" stance. If a project uses `lll`, the line length cap is a *project* policy, not a Style Guide compliance check; don't conflate them.
- **`golines`** — auto-wraps long lines. Aggressive auto-wrapping is exactly what the Guide tells you *not* to do; the Guide wants refactoring, not wrapping.
- **`funlen` / `gocyclo` / `cyclop`** — measure function length / complexity, not line length. Useful tools, but they don't address #08.

### Recommendation

If you want #08 partially automated in CI, a *soft* line-length warning at a generous threshold (say 160-200 columns) can be useful as a **refactor prompt** — it tells the author "this line is long enough that you should consider whether a refactor would help" without enforcing a wrap. Combine with `gofumpt` for the wrapping-style anti-patterns it does catch.

The judgment-heavy half of #08 — *"is this line long because it's been wrapped, or long because the work it does is genuinely irreducible?"* — stays with the reviewer (or this skill's subagent). No linter answers that.

## When NOT to apply

- **`gofmt` already wrapped it** — accept the formatter's choice. Composite literals where each field is on its own line, struct tags that overflow, etc.
- **Generated code** — readability targets the generator's contract, not the human reviewer.
- **Long string constant a tool consumes verbatim** — e.g., embedded SQL, embedded shell, embedded regex. Keep on one line unless `gofmt`/the language requires otherwise.
- **`//go:` directives, `//nolint:…` comments, `// +build` tags** — these often must be on a single physical line; the rule about "no fixed line length" applies.

## Suggested finding phrasing

- "URL string `\"https://...\" + \"...\"` split across lines — Style Guide #08 says do not split strings to fit width; keep on one line."
- "Function signature wrapped before `{` — Style Guide #08 forbids splitting before an indentation change; refactor by reducing parameters or grouping into an Options struct."
- "`if a && b && c {` wrapped mid-expression — extract a named boolean variable rather than wrapping."
- "Method chain `.Foo().Bar().Baz()` split mid-chain — introduce intermediate `:=` variables that name each step."
- "`fmt.Errorf(...)` line wrapped at column 100 — refactor (helper, named args) rather than wrap the literal."

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Line length section (the canonical text quoted above)
