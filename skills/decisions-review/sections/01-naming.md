---
section: "Naming"
number: 1
category: actionable
rules:
  - underscores
  - package-names
  - receiver-names
  - constant-names
  - initialisms
  - getters
  - variable-names
  - single-letter-variable-names
  - repetition (with sub-rules: package-vs-exported-symbol, variable-vs-type, external-vs-local)
sources:
  - https://google.github.io/styleguide/go/decisions#naming
---

# Section 01 — Naming

## What this section covers

Concrete, opinionated decisions for naming things in Go. Where the [Style Guide's `#09 Naming`](../../guide-review/guidelines/09-naming.md) sets the principles (no repetition, context-aware, no concept restating), this section is the **mechanics** Google readability mentors actually enforce: which letters to use, how long, when to drop a `Get`, when an initialism stays uppercase, etc.

> [!IMPORTANT]
> All references to `MixedCaps` in this section ride on the Style Guide's `#07 MixedCaps` rule. Initialism casing in particular (this section's `#initialisms`) is the most-violated detail and should be checked alongside `#07`.

## #underscores — Underscores

> Names in Go should in general not contain underscores.

Three exceptions:

1. Package names that are **only imported by generated code** may contain underscores.
2. **`Test`/`Benchmark`/`Example`** function names within `*_test.go` files may include underscores (the convention is `TestFoo_BarSubcase`).
3. Low-level libraries that interoperate with the OS or cgo may reuse identifiers (as `syscall` does). Rare in most codebases.

**Note:** Source filenames are not Go identifiers — they may contain underscores (`http_test.go`, `parse_url.go`).

### Smells

- Snake-case identifiers in regular code (`user_id`, `parse_url`) → use `userID`, `parseURL`.
- Test helpers named with underscores outside the `Test*_*` convention.
- A package named `my_pkg` that's hand-imported (not generated) — must rename at import or fix the package name.

---

## #package-names — Package names

> In Go, package names must be concise and use only lowercase letters and numbers (e.g., `k8s`, `oauth2`). Multi-word package names should remain unbroken and in all lowercase (e.g., `tabwriter` instead of `tabWriter`, `TabWriter`, or `tab_writer`).

Three additional rules:

1. **Avoid shadow-prone names.** `usercount` is better than `count` because `count` is a common local variable. (See [`best-practices#shadowing`](https://google.github.io/styleguide/go/best-practices#shadowing).)
2. **No underscores.** If you must import a package with an underscore (usually generated/third-party), rename at import time.
3. **Generated-code exception.** Package names imported only by generated code may contain underscores.

### Smells

- `tabWriter`, `tab_writer`, or `TabWriter` as a package name → must be `tabwriter`.
- Package named `util`, `common`, `helpers`, `lib` — uninformative; not strictly forbidden by this rule but always worth questioning.
- Package named with a verb (`compute`, `process`) when a noun would do (`compute`, package; or move logic into a typed home).
- Package name that's commonly used as a local variable (`count`, `time`, `name`, `list`).

---

## #receiver-names — Receiver names

Receiver variable names **must** be:

- **Short** (usually one or two letters in length)
- **Abbreviations for the type itself**
- **Applied consistently** to every receiver for that type (don't mix `t Tray` and `tr Tray` across methods)
- **Not an underscore**; omit the name if it is unused

| Long Name | Better Name |
|---|---|
| `func (tray Tray)` | `func (t Tray)` |
| `func (info *ResearchInfo)` | `func (ri *ResearchInfo)` |
| `func (this *ReportWriter)` | `func (w *ReportWriter)` |
| `func (self *Scanner)` | `func (s *Scanner)` |

### Smells

- `this`, `self`, `me` as a receiver — pick a type-derived abbreviation.
- Different receiver names across methods of the same type — pick one and use it everywhere.
- Long receiver names like `tray` when `t` would do.
- A named receiver (`func (t Tray) Foo()`) where the receiver is unused — omit the name (`func (Tray) Foo()`).
- `_` as the receiver name (when unused) — drop the name entirely instead.

---

## #constant-names — Constant names

> Constant names must use [MixedCaps](https://google.github.io/styleguide/go/guide#mixed-caps) like all other names in Go. (Exported constants start with uppercase, while unexported constants start with lowercase.) This applies even when it breaks conventions in other languages.

> Constant names should not be a derivative of their values and should instead explain what the value denotes.

```go
// Good:
const MaxPacketSize = 512

const (
    ExecuteBit = 1 << iota
    WriteBit
    ReadBit
)
```

> Do not use non-MixedCaps constant names or constants with a `K` prefix.

```go
// Bad:
const MAX_PACKET_SIZE = 512
const kMaxBufferSize = 1024
const KMaxUsersPergroup = 500
```

> Name constants based on their role, not their values. If a constant does not have a role apart from its value, then it is unnecessary to define it as a constant.

```go
// Bad:
const Twelve = 12

const (
    UserNameColumn = "username"
    GroupColumn    = "group"
)
```

(Both bad cases above: `Twelve` adds nothing over `12`; `UserNameColumn = "username"` restates the value.)

### Smells

- `SCREAMING_SNAKE_CASE` constants — convert to `MixedCaps`.
- `K`-prefixed constants (`KMaxFoo`, `kMaxBar`) — Google C++ style, not Go.
- A constant named for its literal value (`Twelve = 12`, `Empty = ""`).
- A constant whose value-as-string equals its name (`UserNameColumn = "username"`) — usually means the constant has no semantic role beyond being the literal.

---

## #initialisms — Initialisms

> Words in names that are initialisms or acronyms (e.g., `URL` and `NATO`) should have the same case. `URL` should appear as `URL` or `url` (as in `urlPony`, or `URLPony`), never as `Url`.

> As a general rule, identifiers (e.g., `ID` and `DB`) should also be capitalized similar to their usage in English prose.

> In names with multiple initialisms (e.g. `XMLAPI` because it contains `XML` and `API`), each letter within a given initialism should have the same case, but each initialism in the name does not need to have the same case.

> In names with an initialism containing a lowercase letter (e.g. `DDoS`, `iOS`, `gRPC`), the initialism should appear as it would in standard prose, unless you need to change the first letter for the sake of [exportedness](https://golang.org/ref/spec#Exported_identifiers). In these cases, the entire initialism should be the same case (e.g. `ddos`, `IOS`, `GRPC`).

### Quick reference

| Initialism | Exported | Unexported (when leading) |
|---|---|---|
| URL | `URL`, `BaseURL` | `url`, `baseURL` |
| HTTP | `HTTP`, `HTTPClient` | `http`, `httpClient` |
| ID | `ID`, `UserID` | `id`, `userID` |
| API | `API`, `APIServer` | `api`, `apiServer` |
| DB | `DB`, `UserDB` | `db`, `userDB` |
| XML+API | `XMLAPI` (each unit consistent; mix case of units OK) | `xmlAPI` |
| DDoS | `DDoS` (mixed-case in prose) | `ddos` (all lowercase) |
| iOS | `IOS` (must change first letter for export) | `ios` |
| gRPC | `GRPC` (must change first letter for export) | `grpc` |

### Smells

- `Url`, `Http`, `Json`, `Xml`, `Id`, `Api`, `Db`, `Cpu`, `Io`, `Tls`, `Sql`, `Uuid`, `Rpc` — convert to all-uppercase as a unit.
- `userId`, `userIdString`, `httpClient` (with lowercase `http` mid-word but uppercase elsewhere) — initialism casing must be consistent within the unit.
- `iOSClient`, `gRPCServer`, `DDoSAttack` mid-name — fine in prose-style, but verify this matches the package's other casing.

---

## #getters — Getters

> Function and method names should not use a `Get` or `get` prefix, unless the underlying concept uses the word 'get' (e.g. an HTTP GET).

> Prefer starting the name with the noun directly, for example use `Counts` over `GetCounts`.

> If the function involves performing a complex computation or executing a remote call, a different word like `Compute` or `Fetch` can be used in place of `Get`, to make it clear to a reader that the function call may take time and could block or fail.

### Decision tree

| Function does... | Name |
|---|---|
| Returns a stored field | drop the prefix: `Counts()`, not `GetCounts()` |
| Performs a meaningful computation | `Compute*`, `Calculate*`, `Build*` |
| Hits the network / DB / disk | `Fetch*`, `Load*`, `Read*` |
| Wraps an HTTP GET (or `os.Getenv`-style domain "get") | `Get*` is OK — domain matches |

### Smells

- `GetUser()`, `GetID()`, `GetConfig()` returning a struct field → drop `Get`.
- `GetUserFromDatabase()` → `LoadUser()` or `FetchUser()` (signal that this is not free).
- `getFoo()` (unexported) where `foo()` would do — same rule applies.
- `Set*` is **not** banned by this rule — only `Get`. (Setters are fine when present.)

---

## #variable-names — Variable names

> The general rule of thumb is that the length of a name should be proportional to the size of its scope and inversely proportional to the number of times that it is used within that scope.

Scope size guide (from the doc):

| Scope size | Lines | Name length |
|---|---|---|
| Small | 1–7 | one or two words; sometimes a single character |
| Medium | 8–15 | one word; sometimes two |
| Large | 15–25 | multi-word; descriptive |
| Very large | >25 (more than a page) | multi-word, fully descriptive |

> A name that might be perfectly clear (e.g., `c` for a counter) within a small scope could be insufficient in a larger scope and would require clarification to remind the reader of its purpose further along in the code.

> The name of a local variable should reflect what it contains and how it is being used in the current context, rather than where the value originated.

> Single-word names like `count` or `options` are a good starting point. Additional words can be added to disambiguate similar names, for example `userCount` and `projectCount`.

> Do not simply drop letters to save typing. For example `Sandbox` is preferred over `Sbx`, particularly for exported names.

> Omit types and type-like words from most variable names. For a number, `userCount` is a better name than `numUsers` or `usersInt`. For a slice, `users` is a better name than `userSlice`.

> It is acceptable to include a type-like qualifier if there are two versions of a value in scope, for example you might have an input stored in `ageString` and use `age` for the parsed value.

> Omit words that are clear from the surrounding context.

### Smells

- Long variable in a tiny scope (`userServiceClient` in a 3-line block) — shrink.
- Single-letter variable in a 60-line function (`x` everywhere) — expand.
- Letter-dropped abbreviations (`Sbx`, `Mgr`, `Cfg`, `Svc`, `Cnt`) — spell out.
- Type-suffixed names (`numUsers`, `userSlice`, `nameString`, `usersInt`) — drop the suffix unless two forms coexist.
- Names tied to where the value *came from* rather than what it *is now* (`fromAPIResp`, `dbResult` used as the parsed business object).

---

## Single-letter variable names

> Single-letter variable names can be a useful tool to minimize repetition, but can also make code needlessly opaque.

> Limit their use to instances where the full word is obvious and where it would be repetitive for it to appear in place of the single-letter variable.

> For a method receiver variable, a one-letter or two-letter name is preferred. (See `#receiver-names`.)

> Using familiar variable names for common types is often helpful: `r` for an `io.Reader` or `*http.Request`; `w` for an `io.Writer` or `http.ResponseWriter`.

> Single-letter identifiers are acceptable as integer loop variables, particularly for indices (e.g., `i`) and coordinates (e.g., `x` and `y`).

> Abbreviations can be acceptable loop identifiers when the scope is short, for example `for _, n := range nodes { ... }`.

### Acceptable single letters (community convention)

| Letter | Context |
|---|---|
| `r` | `io.Reader`, `*http.Request` |
| `w` | `io.Writer`, `http.ResponseWriter` |
| `t` | `*testing.T`, `time.Time`, `Tray`-style receiver |
| `b` | `*testing.B`, `[]byte`, byte buffer |
| `i`, `j`, `k` | loop indices |
| `x`, `y`, `z` | coordinates |
| `n` | a count / loop limit / generic numeric |
| `s` | string parameter, receiver |
| `err` | error |
| `ctx` | `context.Context` |
| `_` | intentionally discarded |

### Smells

- `x` outside a coordinate / math context, in a non-tight scope.
- Single-letter receivers that don't match the type's first letter (`a Tray`).
- `for _, x := range somethingMeaningful { … }` — `x` here loses signal; `for _, n := range nodes { … }` is better.

---

## #repetition — Repetition

> A piece of Go source code should avoid unnecessary repetition. One common source of this is repetitive names, which often include unnecessary words or repeat their context or type. Code itself can also be unnecessarily repetitive if the same or a similar code segment appears multiple times in close proximity.

The Decisions doc breaks this into three sub-cases:

### #repetition / Package vs. exported symbol name

> When naming exported symbols, the name of the package is always visible outside your package, so redundant information between the two should be reduced or eliminated. If a package exports only one type and it is named after the package itself, the canonical name for the constructor is `New` if one is required.

| Repetitive | Better |
|---|---|
| `widget.NewWidget` | `widget.New` |
| `widget.NewWidgetWithName` | `widget.NewWithName` |
| `db.LoadFromDatabase` | `db.Load` |
| `goatteleportutil.CountGoatsTeleported` | `gtutil.CountGoatsTeleported` or `goatteleport.Count` |
| `myteampb.MyTeamMethodRequest` | `mtpb.MyTeamMethodRequest` or `myteampb.MethodRequest` |

### #repetition / Variable name vs. type

> The compiler always knows the type of a variable, and in most cases it is also clear to the reader what type a variable is by how it is used. It is only necessary to clarify the type of a variable if its value appears twice in the same scope.

| Repetitive | Better |
|---|---|
| `var numUsers int` | `var users int` |
| `var nameString string` | `var name string` |
| `var primaryProject *Project` | `var primary *Project` |

When two forms coexist:

```go
// Good:
limitRaw := r.FormValue("limit")
limit, err := strconv.Atoi(limitRaw)
```

```go
// Good:
limitStr := r.FormValue("limit")
limit, err := strconv.Atoi(limitStr)
```

### #repetition / External context vs. local names

> Names that include information from their surrounding context often create extra noise without benefit. The package name, method name, type name, function name, import path, and even filename can all provide context that automatically qualifies all names within.

```go
// Bad:
// In package "ads/targeting/revenue/reporting"
type AdsTargetingRevenueReport struct{}

func (p *Project) ProjectName() string
```

```go
// Good:
// In package "ads/targeting/revenue/reporting"
type Report struct{}

func (p *Project) Name() string
```

```go
// Bad:
// In package "sqldb"
type DBConnection struct{}
```

```go
// Good:
// In package "sqldb"
type Connection struct{}
```

```go
// Bad:
// In package "ads/targeting"
func Process(in *pb.FooProto) *Report {
    adsTargetingID := in.GetAdsTargetingID()
}
```

```go
// Good:
// In package "ads/targeting"
func Process(in *pb.FooProto) *Report {
    id := in.GetAdsTargetingID()
}
```

### A combined example

Repetition should be evaluated in context, not in isolation. The doc gives this:

```go
// Bad:
func (db *DB) UserCount() (userCount int, err error) {
    var userCountInt64 int64
    if dbLoadError := db.LoadFromDatabase("count(distinct users)", &userCountInt64); dbLoadError != nil {
        return 0, fmt.Errorf("failed to load user count: %s", dbLoadError)
    }
    userCount = int(userCountInt64)
    return userCount, nil
}
```

```go
// Good:
func (db *DB) UserCount() (int, error) {
    var count int64
    if err := db.Load("count(distinct users)", &count); err != nil {
        return 0, fmt.Errorf("failed to load user count: %s", err)
    }
    return int(count), nil
}
```

The bad version repeats `UserCount`, `userCount`, `userCountInt64`, `dbLoadError`, `LoadFromDatabase` — every one of which is redundant given the function name, the package, and `err`'s well-known role.

### Smells (across all repetition sub-rules)

- `pkg.NewPkgName()` — drop the type from the constructor.
- `Pkg.PkgMethod()` — drop the type from the method.
- `var fooString string`, `var fooInt int` — drop the type suffix unless two forms coexist.
- Field/var named with the package's own name (`adsTargetingID` in `ads/targeting`).
- Custom error names that re-state context (`dbLoadError` instead of `err`) — Go convention is `err` always.
- Long stuttering type names (`SQLDBConnection`) when the package already qualifies them.

---

## Consolidated review checklist (all 12 rules)

- [ ] No underscores in identifiers (except `Test*_*`, generator-only packages, syscall).
- [ ] Package names: concise, lowercase, no underscores, not shadow-prone.
- [ ] Receiver names: 1–2 letters, derived from type, consistent across methods, no `this`/`self`.
- [ ] Constants: `MixedCaps`, no `K`-prefix, no `SCREAMING_SNAKE`, name encodes role not value.
- [ ] Initialisms: same case as a unit (`URL`, not `Url`); `iOS`/`gRPC`/`DDoS` follow prose-style except when re-cased for exportedness.
- [ ] No `Get`/`get` prefix on field-returning accessors; use `Compute`/`Fetch` for expensive ones.
- [ ] Variable name length is proportional to scope; no letter-dropped abbreviations; type-suffix only when two forms coexist.
- [ ] Single-letter names confined to short loops, receivers, or community-conventional types (`r`, `w`, `i`, `n`, `ctx`, `err`).
- [ ] No package/symbol stutter at the call site (`widget.NewWidget` → `widget.New`).
- [ ] No type-suffix on var names where the type is obvious (`numUsers` → `users`).
- [ ] No package-context restatement in local names (`adsTargetingID` in `ads/targeting`).

## Suggested finding phrasing

- "`UserService.GetUser()` returns a struct field — Decisions §getters: drop `Get`, name the method `User()`."
- "`MAX_RETRIES` constant — Decisions §constant-names: use `MaxRetries` (or `maxRetries` if unexported)."
- "`HttpClient` field on `Server` struct — Decisions §initialisms: initialism stays uppercase as a unit; rename to `HTTPClient`."
- "`func (this *Scanner) Scan()` — Decisions §receiver-names: receivers must derive from the type; use `s` (and apply consistently across all methods on `*Scanner`)."
- "`var nameString string` — Decisions §repetition / Variable name vs. type: drop the `String` suffix; use `name`. Add `Raw` / `Str` only if a parsed form coexists in scope."
- "`widget.NewWidget(name)` — Decisions §repetition / Package vs. exported symbol: rename constructor to `New`."
- "`type AdsTargetingRevenueReport` in package `ads/targeting/revenue/reporting` — Decisions §repetition / External context vs. local names: rename to `Report`."
- "Receiver `tray` for type `Tray` — Decisions §receiver-names: 1–2 letters; use `t`."
- "Constant `Twelve = 12` — Decisions §constant-names: a constant equal to its own value adds no role; either give it a meaningful name (`MaxRetries`) or remove the constant."

## Sources

- <https://google.github.io/styleguide/go/decisions#naming> — Google Go Style Decisions, Naming section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/guide#mixed-caps> — Style Guide §MixedCaps (referenced by `#constant-names` and tied to `#initialisms`)
- <https://google.github.io/styleguide/go/best-practices#shadowing> — Best Practices §shadowing (referenced by `#package-names` for shadow-prone names)
- <https://golang.org/ref/spec#Method_declarations> — Go spec on method declarations (receiver syntax)
- <https://golang.org/ref/spec#Exported_identifiers> — Go spec on exported identifiers (capitalization rule)
