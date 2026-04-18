---
guideline: "Naming"
number: 9
category: actionable
sources:
  - https://google.github.io/styleguide/go/guide
  - https://google.github.io/styleguide/go/decisions#naming
  - https://google.github.io/styleguide/go/decisions#repetition
  - https://testing.googleblog.com/2017/10/code-health-identifiernamingpostforworl.html
---

# #09 — Naming

## What it requires

The Style Guide's Naming subsection is short — it states three principles and defers the detailed mechanics to Style Decisions:

> Naming is more art than science. In Go, names tend to be somewhat shorter than in many other languages, but the same [general guidelines](https://testing.googleblog.com/2017/10/code-health-identifiernamingpostforworl.html) apply.

> Names should:
> - Not feel repetitive when they are used
> - Take the context into consideration
> - Not repeat concepts that are already clear

> You can find more specific guidance on naming in [decisions](https://google.github.io/styleguide/go/decisions#naming).

So this rule has three checks at the Style Guide level (the deeper, decision-level mechanics — initialisms, package names, receiver names, getters, etc. — are covered by `decisions-review`):

1. **No repetition at the use site.** A name shouldn't make `pkg.Name` read awkwardly.
2. **Context-aware.** Same concept gets different name lengths in different scopes.
3. **No restating what's already clear.** The type, the package, or the surrounding code may already convey what a name would otherwise duplicate.

## What to look for (code smells)

### Repetition (use-site stutter)

The number-one Style Guide naming smell — the *use site* shouldn't read with the package name (or owning type name) twice:

- `widget.NewWidget()` → `widget.New()`
- `http.HTTPClient` → `http.Client`
- `user.User` (struct) called as `user.User{}` everywhere → consider `user.Account` or rename the package
- `errors.ErrNotFound` → either `errors.NotFound` (if Sentinel) or live with the convention; flag mainly when it really stutters
- `config.ConfigOption` → `config.Option`
- `service.UserService` → `service.User` (the package name supplies "service")
- `db.DBConnection` → `db.Connection`

The general rule: read the name *as it appears at the call site*. If the package or owning type is repeated, trim.

### Context insensitivity

The Style Guide implicitly endorses a known Go idiom (made explicit in Decisions): **scope-proportional name length** — short scope → short name, large scope → longer descriptive name.

- A loop variable `userServiceClient` in a 3-line `for` — too long for the scope. `c` or `cli` is fine.
- A package-level variable named `c` or `x` — too short for the scope. `client`, `defaultClient` is needed.
- A struct field named `i` or `n` — usually too short; the field outlives any function call site.
- A method receiver named `userServiceManager` — Go convention is 1-3 letters (`s`, `m`, `usm` etc.). Decisions covers this in detail; flag here only the most extreme cases.

### Restating already-clear concepts

The name shouldn't tell you what the type/package/method already tells you:

- `userMap map[string]User` → `users map[string]User` (the type already says "map")
- `func (u *User) GetUser() *User` → `func (u *User) Get() *User` (or, if it's just a getter, drop it entirely; see Decisions §getters)
- `errCount int` (`int` is clear) — fine. But `userCountInt int` — drop the suffix.
- `xmlParser` in package `xml` → just `Parser`
- `httpHandlerFunc` field of type `http.HandlerFunc` → `handler` (the type encodes the rest)
- `isActiveBool bool` → `isActive bool` (or `active bool` — the `is` prefix is sometimes contested; Decisions has a take)

### Other smells

- **Numerical or temporal suffixes** when the older version isn't gone (`HandlerV2`, `Handler2024`) — usually a sign of incomplete migration. Decide whether to delete the old one or rename both with intent.
- **Generic placeholders** as final names: `data`, `info`, `util`, `helper`, `common`, `manager`, `processor`. Each almost always means "I haven't decided what this *is*."
- **Single-character names outside small scopes** (loop counter `i`, error `err`, context `ctx`, receiver `s` are fine; `x` across a 60-line function is not).
- **Magic-number constants without a named constant** — not strictly a naming rule but often filed here. If the constant has a meaning, give it a name.

## Good patterns

```go
// Use-site reads cleanly — no stutter
client := http.NewClient()         // not http.NewHTTPClient()
opts   := config.Default()         // not config.DefaultConfig()
notFound := errors.New("missing")  // sentinel via package convention

// Scope-proportional name length
for i, line := range lines {       // short scope, short names
    process(line)
}

// vs. package-level — descriptive
var defaultRetryDelay = 5 * time.Second

// Field name doesn't restate the type
type Server struct {
    handler http.HandlerFunc       // not handlerFunc, not httpHandler
    started time.Time              // not startedTime
}

// Receiver: short, mnemonic
func (s *Server) Start() error { … }

// Concept-first naming, type-suffix avoided
users := make(map[string]User)     // not userMap
```

## Review checklist

- [ ] Reading the name at the *call site*, does the package or owning type appear twice? (`xml.XMLParser` → `xml.Parser`)
- [ ] Is the name's length proportional to its scope? (1-letter for tight loops, multi-word for package-level)
- [ ] Does the name restate what the type / package / method already conveys? (`userMap` for `map[string]User`)
- [ ] Are generic words (`data`, `info`, `util`, `helper`, `common`, `manager`) standing in for an undecided concept?
- [ ] Are migration suffixes (`V2`, `2024`, `New`, `Old`) still hanging around after the migration?
- [ ] Do `Get*` getters that just return a field exist? (Decisions §getters: drop `Get`.)
- [ ] Are receiver names 1–3 letters and consistent across all methods on the type?
- [ ] Are local variables named for *what they hold*, not *what type they are*? (`requestStart`, not `t`; `errCount`, not `i`)

## When NOT to apply

- **Public API stability.** A widely-imported package with stuttering names (`bytes.Buffer`, `bufio.Reader`) is a tax that's been paid; renaming would break callers. Note the smell, don't push the rename.
- **Generated code.** Names follow the source spec (proto, OpenAPI, etc.). Flag the spec, not the generated Go.
- **Domain terms** that legitimately repeat. `oauth.OAuth2Token` is correct because `OAuth2Token` is the protocol's name. Same for `xml.XMLName` (a known proto-style sentinel).
- **Test functions** — `TestParseURL_InvalidScheme` repeats `Test` and the function under test by convention. That's not stutter; it's required structure.

## Suggested finding phrasing

- "`widget.NewWidget()` — Style Guide #09 (no repetition): use `widget.New()` at the call site."
- "`config.ConfigOption` — `Option` reads cleaner from the `config` package."
- "`userMap map[string]User` — the type encodes 'map'; rename the variable to `users`."
- "`func (u *User) GetUser() *User` — Decisions §getters: drop `Get`; if it's only returning a field, drop the method too."
- "`data` parameter in `func handle(data []byte)` — pick a concept-bearing name (`payload`, `body`, `frame`)."
- "`HandlerV2` exists alongside `Handler` — finish the migration or rename to encode the actual difference (`StreamingHandler` etc.)."
- "Receiver name `userServiceManager` is unusually long — Go convention is 1–3 letters; use `m` or `usm`."

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Naming section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#naming> — Style Decisions, expanded naming guidance (covered by sibling skill `decisions-review`)
- <https://google.github.io/styleguide/go/decisions#repetition> — Style Decisions, repetition specifics
- <https://testing.googleblog.com/2017/10/code-health-identifiernamingpostforworl.html> — Google Testing Blog, "Identifier Naming" (referenced as the general guidelines)
