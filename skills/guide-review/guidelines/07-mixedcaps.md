---
guideline: "MixedCaps for multi-word identifiers"
number: 7
category: actionable
sources:
  - https://google.github.io/styleguide/go/guide
  - https://go.dev/ref/spec#Exported_identifiers
  - https://google.github.io/styleguide/go/decisions#initialisms
  - https://google.github.io/styleguide/go/decisions#underscores
---

# #07 — MixedCaps for multi-word identifiers

## What it requires

The Style Guide is short and absolute on this:

> Go source code uses `MixedCaps` or `mixedCaps` (camel case) rather than underscores (snake case) when writing multi-word names.

> This applies even when it breaks conventions in other languages.

> For example, a constant is `MaxLength` (not `MAX_LENGTH`) if exported and `maxLength` (not `max_length`) if unexported.

> Local variables are considered [unexported](https://go.dev/ref/spec#Exported_identifiers) for the purpose of choosing the initial capitalization.

In short:

| Identifier kind | Form | Example |
|---|---|---|
| Exported (package level, struct field, method) | `MixedCaps` (initial uppercase) | `MaxLength`, `UserID`, `ParseRequest` |
| Unexported (package level) | `mixedCaps` (initial lowercase) | `defaultTimeout`, `parseHeader` |
| Constants — exported | `MixedCaps` (NOT `SCREAMING_SNAKE`) | `MaxRetries` |
| Constants — unexported | `mixedCaps` | `maxRetries` |
| Local variables | `mixedCaps` (treated as unexported) | `count`, `requestStart` |
| Initialisms | All-caps **as a unit** within MixedCaps | `userID`, `HTTPServer`, `parseURL` |

The initialism rule comes from Style Decisions, not the Guide directly — but it's the single most-violated MixedCaps detail, so it belongs here. Direct from Decisions: `URL`, `HTTP`, `ID`, `JSON`, `XML` etc. stay as a single capitalized unit (`HTTPServer` not `HttpServer`; `userID` not `userId`).

## What to look for (code smells)

- Any identifier with `_` in the middle: `max_length`, `parse_request`, `user_id`. **Exception:** test function suffixes (`TestFoo_BarCase`) are valid Go convention — see Style Decisions.
- `SCREAMING_SNAKE_CASE` constants — common in code ported from C / Python / Java / .env conventions.
- Initialisms in mixed case: `Url`, `Http`, `Json`, `Xml`, `Id`, `Api`, `Cpu`, `Db` — should be `URL`, `HTTP`, `JSON`, `XML`, `ID`, `API`, `CPU`, `DB`.
- Lowercased initialisms at the start of unexported identifiers: this is correct (`urlParser`, `httpClient`, `idGen`) — the leading initialism stays lowercase to mark unexported, but the rest of it stays uppercase as a unit.
- Hungarian-notation prefixes / suffixes (`strName`, `iCount`) — use the type system, not the name.
- Numeric suffixes wedged with underscores: `parser_v2`, `handler_2024` → `parserV2`, `handler2024`.
- Constants spelled out as `MAX_RETRIES`, `DEFAULT_TIMEOUT`, `STATUS_OK` — convert.
- Field names in struct tags **may** legitimately use underscores (e.g., `json:"user_id"`) — that's the wire format, not the Go name. Don't flag tags.
- Generated code that uses non-`MixedCaps` names because the generator targets a different spec (protobuf is fine — generated `User_Id` is OK if the proto field is named that way; flag the *proto definition*, not the generated Go).

## Good patterns

```go
// Exported constants
const (
    MaxRetries     = 3
    DefaultTimeout = 5 * time.Second
)

// Unexported package-level
var defaultClient = http.Client{Timeout: DefaultTimeout}

// Initialisms — exported and unexported
type HTTPServer struct {
    URL    string
    UserID int64
}

func (s *HTTPServer) parseURL(raw string) (*url.URL, error) { … }

// Local variables — always lowerCamel
func handle(req *Request) error {
    requestStart := time.Now()
    parsedURL, err := url.Parse(req.URL)
    …
}

// Acronym at start of unexported name — initialism stays uppercase as a unit,
// but the leading character is lowercased
var idGenerator = …  // OK
var dbConnection = … // OK
```

## Review checklist

- [ ] No identifier contains `_` (excluding test function suffixes like `TestFoo_Bar` and struct-tag wire formats).
- [ ] No constant uses `SCREAMING_SNAKE_CASE`.
- [ ] All recognised initialisms (`URL`, `HTTP`, `JSON`, `XML`, `ID`, `API`, `CPU`, `DB`, `IO`, `TLS`, `SQL`, `UUID`, `RPC`, …) are written as a single uppercase unit, not `Url` / `Http` / etc.
- [ ] Exported names start uppercase; unexported and local names start lowercase.
- [ ] Numeric suffixes are joined without underscores: `parserV2`, not `parser_v2`.
- [ ] Hungarian-style type prefixes/suffixes are absent.
- [ ] Generated-code violations (if any) are traced to the source spec (`.proto`, `.thrift`, etc.), not the Go file.

## When NOT to apply

- **Struct tags** — the tag value (`json:"user_id"`, `db:"created_at"`) is wire format, not a Go name. Don't flag.
- **Test function names with underscore suffix** — `TestParseURL_InvalidScheme` is the standard subtest naming convention. The portion before `_` is still MixedCaps.
- **Generated code with `// Code generated ... DO NOT EDIT.`** — the source-of-truth is the generator's input. Flag the spec, not the generated Go.
- **Cgo / assembly bindings** — when interfacing with C symbols, the Go name often must match a non-MixedCaps external symbol. Add a doc comment explaining and move on.
- **Build-tagged platform shims** — if a name must match a platform constant, follow the platform.

## Suggested finding phrasing

- "`MAX_RETRIES` constant uses SCREAMING_SNAKE — Style Guide #07 requires `MaxRetries` (exported) or `maxRetries` (unexported)."
- "`HttpClient` mixes initialism case — Style Guide #07 + Decisions §initialisms require `HTTPClient`."
- "`parse_url` function — Style Guide #07 requires `parseURL` (lowercase initial, uppercase initialism unit)."
- "`UserId` field — initialism `ID` should be uppercase as a unit: `UserID`."
- "`local_var` in `handle()` — local variables are unexported for capitalization purposes; use `localVar`."
- "`parser_v2` — Style Guide #07: `parserV2`, no underscore."

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, MixedCaps section (the canonical text quoted above)
- <https://go.dev/ref/spec#Exported_identifiers> — Go spec on exported vs unexported (capitalization rule)
- <https://google.github.io/styleguide/go/decisions#initialisms> — Style Decisions on initialism casing (covered by sibling skill `decisions-review`)
- <https://google.github.io/styleguide/go/decisions#underscores> — Style Decisions on underscores in identifiers (test function exception)
