---
section: "Common libraries"
number: 6
category: actionable
rules:
  - flags
  - logging
  - contexts
  - custom-contexts
  - cryptorand
sources:
  - https://google.github.io/styleguide/go/decisions#common-libraries
---

# Section 06 — Common libraries

## What this section covers

Five concrete decisions on standard-library usage: how to define `flag` variables, the use of Google's internal `log` variant, the rules for passing `context.Context`, the absolute prohibition on custom context types, and the requirement to use `crypto/rand` (not `math/rand`) for any keying material.

> [!NOTE]
> Two of these rules (`#flags` and `#logging`) reference Google-internal variants of stdlib packages. For non-Google projects, the same *principles* apply with whichever flag/log library you use (`spf13/pflag`, `urfave/cli`, `slog`, `zap`, etc.). The `#contexts`, `#custom-contexts`, and `#cryptorand` rules are universal — they apply to all Go code regardless of the codebase.

## #flags — Flags

> Go programs in the Google codebase use an internal variant of the standard `flag` package.

> Flag names in Go binaries should prefer to use underscores to separate words, though the variables that hold a flag's value should follow the standard Go name style (mixed caps).

> Specifically, the flag name should be in snake case, and the variable name should be the equivalent name in camel case.

```go
var (
    pollInterval = flag.Duration("poll_interval", time.Minute, "Interval to use for polling.")
)
```

Note the **two different cases on the same line**:
- The flag *name* (the string the user types on the CLI): `"poll_interval"` — snake_case.
- The Go *variable* holding the flag value: `pollInterval` — camelCase, follows `#variable-names`.

> Flags must only be defined in `package main` or equivalent.

> General-purpose packages should be configured using Go APIs, not by punching through to the command-line interface.

> If your flags are global variables, place them in their own `var` group, following the imports section.

### Smells

- Flag named `"pollInterval"` (camelCase) on the CLI — should be `"poll_interval"`.
- Flag-defining variable named `poll_interval` (snake_case in Go) — should be `pollInterval`.
- A flag defined in a non-`main` package (`util/config/flags.go` with `flag.String(...)`) — Decisions forbids this; library packages take Go-API config (struct, functional options, etc.), not CLI flags.
- Flags scattered throughout `main.go` instead of grouped in a single `var (...)` block right after imports.
- Library function reading `os.Args` or doing `flag.Parse()` from inside the library — same anti-pattern as defining flags there.

---

## #logging — Logging

> Go programs in the Google codebase use a variant of the standard `log` package. It has a similar but more powerful interface and interoperates well with internal Google systems.

> **Note:** For abnormal program exits, this library uses `log.Fatal` to abort with a stacktrace, and `log.Exit` to stop without one.

> `log.Info(v)` is equivalent `log.Infof("%v", v)`

> Prefer the non-formatting version when you have no formatting to do.

The actionable bits for non-Google projects:

1. **Use the project's chosen log library consistently.** Don't mix `log` (stdlib), `slog`, `zap`, `logrus`, etc. in the same project.
2. **Prefer non-formatting calls when there's nothing to format.** `log.Info(msg)` over `log.Infof("%v", msg)` — same output, less work.
3. **Distinguish abnormal-exit primitives.** `log.Fatal` (with stacktrace) vs `log.Exit` (without) — pick the one that matches whether a stack helps debugging.

### Smells

- `log.Infof("%v", x)` where `log.Info(x)` is equivalent — drop the format string.
- Multiple log libraries imported in the same package (`stdlog "log"` alongside `slog "log/slog"`).
- `log.Fatal` used inside a library function — unintended exit; libraries should return errors and let `main` decide.
- `os.Exit(1)` inline instead of using the log library's exit primitive (prevents log buffering from flushing).
- `panic(...)` used for normal error paths — see `#dont-panic` (in section 05, when written).

---

## #contexts — Contexts

> `context.Context` is always the first parameter in function signatures.

Exceptions:

1. **HTTP handlers** — context comes from `req.Context()`.
2. **Streaming RPC methods** — context comes from the stream.
3. **Test functions** — context comes from `(testing.TB).Context()`.
4. **Other entrypoint functions** — use `context.Background()`.

> Do not add a context member to a struct type. Instead, add a context parameter to each method on the type that needs to pass it along.

> Do not create custom context types or use interfaces other than `context.Context` in function signatures. There are no exceptions to this rule.

### The four-rule summary

| Rule | Bad | Good |
|---|---|---|
| `ctx` is first param | `func Do(id int, ctx context.Context)` | `func Do(ctx context.Context, id int)` |
| `ctx` is named `ctx` | `func Do(c context.Context)` | `func Do(ctx context.Context)` |
| No `ctx` in struct fields | `type Server struct { ctx context.Context }` | `func (s *Server) Handle(ctx context.Context, ...)` |
| No custom context types | `func Do(ctx MyContext)` | `func Do(ctx context.Context)` |

### Smells

- `context.Context` not the first parameter (`func Foo(id int, ctx context.Context, opts Options)`).
- Context parameter named anything other than `ctx` (`c`, `cx`, `context_`, `theContext`).
- `ctx context.Context` as a struct field on a long-lived struct (`Server`, `Worker`, `Client`) — pass it per-call instead.
- A handler function that receives `ctx` but uses `context.Background()` instead — defeats cancellation/timeout.
- A library function that calls `context.WithCancel(context.Background())` instead of `context.WithCancel(ctx)` — drops the caller's deadlines.
- A function with no `ctx` parameter that does I/O or RPC — add `ctx context.Context` as the first parameter.

---

## #custom-contexts — Custom contexts

> Do not create custom context types or use interfaces other than `context.Context` in function signatures. There are no exceptions to this rule.

> Imagine if every team had a custom context. Every function call from package `p` to package `q` would have to determine how to convert a `p.Context` to a `q.Context`, for all pairs of packages `p` and `q`.

> If you have application data to pass around, put it in a parameter, in the receiver, in globals, or in a `Context` value if it truly belongs there.

The doc is unusually absolute on this — "no exceptions." If you need to thread state, the four allowed mechanisms are:

1. **Function parameter** (preferred for most data).
2. **Receiver field** (for data tied to a long-lived object).
3. **Global** (for true singletons — config, registries).
4. **`context.WithValue`** (only for data that genuinely belongs in the context — request-scoped values like trace IDs, auth tokens).

### Smells

- Any type named `Context`, `RequestContext`, `AppContext`, etc. used in function signatures in place of `context.Context`.
- An interface that "wraps" `context.Context` to add domain methods (`type DomainContext interface { context.Context; UserID() string }`) — use plain `context.Context` and `ctx.Value(userIDKey)` for the data, or pass the data as a separate argument.
- A type alias (`type AppContext = context.Context`) — accomplishes nothing, adds confusion.
- Conversion helpers (`func (ac AppContext) ToContext() context.Context`) — strong sign of a custom context that needs to be removed.

---

## #cryptorand — `crypto/rand`

> Do not use package `math/rand` to generate keys, even throwaway ones.

> If unseeded, the generator is completely predictable. Seeded with `time.Nanoseconds()`, there are just a few bits of entropy.

> Instead, use `crypto/rand`'s Reader, and if you need text, print to hexadecimal or base64.

```go
// Good:
import (
    "crypto/rand"
    "fmt"
)

func Key() string {
    buf := make([]byte, 16)
    if _, err := rand.Read(buf); err != nil {
        log.Fatalf("Out of randomness, should never happen: %v", err)
    }
    return fmt.Sprintf("%x", buf)
}
```

The rule applies to **any keying material** — session tokens, API keys, password reset tokens, nonces, IVs, even values used in tests if they ever touch real systems.

### What `math/rand` is still fine for

- Picking a random element for **non-security** purposes (load balancing, jitter, sample selection).
- Test fixtures that don't simulate real keys.
- Game / simulation randomness.

The boundary: anything that's used to **prove identity, authorize access, or be unguessable to an adversary** must come from `crypto/rand`.

### Smells

- `math/rand.Intn(...)` or `math/rand.Read(...)` used to generate a token, key, password, or session ID.
- `time.Now().UnixNano()` used as a seed for a key generator (and the result then used as a key).
- "Throwaway" keys generated by `math/rand` — Decisions explicitly bans this; "throwaway" is not a justification.
- `crypto/rand.Read` called without checking the error — the panic risk exists; check the error and log/abort.
- Use of UUIDv1 (time-based) for tokens needing unguessability — UUIDv4 (random) backed by `crypto/rand` is the correct choice.

---

## Consolidated review checklist (all 5 rules)

- [ ] All flag names use snake_case (`"poll_interval"`); the Go variables holding them use camelCase (`pollInterval`).
- [ ] Flags defined only in `package main` (or test main); library packages take config via Go APIs.
- [ ] Flag declarations grouped in a single `var (…)` block following the imports section.
- [ ] Logging uses one library consistently; prefer non-format calls (`log.Info(msg)`) when there's nothing to format.
- [ ] Libraries return errors instead of calling `log.Fatal` / `os.Exit`.
- [ ] `context.Context` is the first parameter, named `ctx`, in every function that needs it.
- [ ] No `context.Context` field on long-lived structs.
- [ ] No custom context types; no interfaces wrapping `context.Context` in function signatures.
- [ ] All keying material (tokens, keys, nonces, session IDs) generated via `crypto/rand`, not `math/rand`.
- [ ] `crypto/rand.Read` errors are checked and handled.

## Suggested finding phrasing

- "Flag `\"pollInterval\"` defined with camelCase name — Decisions §flags: CLI flag names are snake_case (`\"poll_interval\"`), only the Go variable is camelCase."
- "Flag `flag.String(\"db_url\", ...)` defined in package `internal/db` — Decisions §flags: flags must be defined in `package main`; expose a Go API (struct field or functional option) instead."
- "`log.Infof(\"%v\", err)` — Decisions §logging: prefer `log.Info(err)` when there's no formatting to do."
- "`log.Fatal(...)` inside library function `internal/parser.Parse` — Decisions §logging: libraries should return errors; only `main` decides to exit."
- "`func Fetch(id int, ctx context.Context)` — Decisions §contexts: `ctx context.Context` must be the first parameter."
- "`type Server struct { ctx context.Context }` — Decisions §contexts: do not store context in struct fields; pass it per-method."
- "`func Do(ctx AppContext)` — Decisions §custom-contexts: no custom context types in function signatures; use `context.Context`."
- "`token := math/rand.Int63()` — Decisions §cryptorand: keying material must come from `crypto/rand`, never `math/rand`."

## Sources

- <https://google.github.io/styleguide/go/decisions#common-libraries> — Google Go Style Decisions, Common libraries section (the canonical text quoted above)
- <https://pkg.go.dev/flag> — `flag` package (stdlib)
- <https://pkg.go.dev/context> — `context` package (universal `#contexts` rule)
- <https://pkg.go.dev/crypto/rand> — `crypto/rand` package (`#cryptorand`)
- <https://pkg.go.dev/log/slog> — `slog` (the modern stdlib structured logger; relevant if not using Google's internal log variant)
