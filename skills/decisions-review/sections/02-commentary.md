---
section: "Commentary"
number: 2
category: actionable
rules:
  - comment-line-length
  - doc-comments
  - comment-sentences
  - examples
  - named-result-parameters
  - package-comments
sources:
  - https://google.github.io/styleguide/go/decisions#commentary
---

# Section 02 — Commentary

## What this section covers

How to write Go comments — line length, doc-comment requirements for exported names, sentence shape, runnable `Example*` functions, named result parameters as a documentation tool, and the mandatory package comment.

> [!IMPORTANT]
> This section pairs with the Style Guide's `#01 Clarity` ("comments explain why, not what") and `#06 Formatting` (comments are part of the source). Doc-comment formatting is also enforced by `gofmt -s` / `gofumpt` to some extent — see `#06 Formatting (gofmt)`.

## #comment-line-length — Comment line length

> There is no fixed line length for comments in Go.

> Long comment lines should be wrapped to ensure that source is readable in tools which do not perform automatic wrapping of comment lines.

> If you are uncertain where to wrap, 80 or 100 columns are common choices. However, this is not a hard cut-off; there are situations where breaking a long literal text is harmful.

> There is no requirement for the specific column width at which wrapping occurs. Aim to be consistent within a file.

The doc shows two patterns: a **good** wrap that fits narrow screens, and a **bad** unwrapped giant single-line comment that creates a poor reader experience. The principle: wrap for readability, but don't over-wrap content (URLs, log messages, identifiers) that must stay intact.

### Smells

- A multi-paragraph comment as one giant single line.
- A single file mixing 80-column wrapping in some comments and 200-column in others — pick one.
- A wrapped URL or copy-pasted error message broken across lines (defeats search/grep).
- Comments with mid-word breaks (`// the configura-`) — never; wrap at word boundaries.

---

## #doc-comments — Doc comments

> All top-level exported names must have doc comments, as should unexported type or function declarations with unobvious behavior or meaning.

> These comments should be full sentences that begin with the name of the object being described.

> Doc comments appear in Godoc and are surfaced by IDEs, and therefore should be written for anyone using the package.

> A documentation comment applies to the following symbol, or the group of fields if it appears in a struct.

And the consistency rule for unexported code:

> If you have doc comments for unexported code, follow the same custom as if it were exported.

### Required form

```go
// Sandbox creates a new sandbox in the user's home directory and
// returns a handle for further operations.
func Sandbox(opts Options) (*Box, error) { … }
```

Things to note:
- Begins with the symbol name (`Sandbox …`), not `// Creates …`.
- Full sentence, capitalized, ends with period.
- No blank line between comment and `func` / `type` / `var`.

### Smells

- Exported symbol with no doc comment (`func PublicThing() error { … }` — no comment above it).
- Doc comment that doesn't start with the symbol name (`// Creates a sandbox …` instead of `// Sandbox creates …`).
- Blank line between comment and the symbol it documents.
- Trailing line-comment used as a doc comment (`func Foo() // does the foo`) — must be a leading block comment.
- Auto-generated stubs left in (`// TODO: comment me` on exported APIs).
- Different conventions on unexported vs exported within the same file (commented exports, uncommented unexported types with non-obvious behaviour).

---

## #comment-sentences — Comment sentences

> Comments that are complete sentences should be capitalized and punctuated like standard English sentences.

> Comments that are sentence fragments have no such requirements for punctuation or capitalization.

> Documentation comments should always be complete sentences, and as such should always be capitalized and punctuated.

> Simple end-of-line comments (especially for struct fields) can be simple phrases that assume the field name is the subject.

### Examples

```go
// Sandbox creates a new sandbox.       // doc comment: full sentence
func Sandbox() (*Box, error) { … }

type Server struct {
    addr string  // bind address, e.g. "localhost:8080"  // fragment OK; subject is `addr`
    tls  bool    // enable TLS                            // fragment OK
}

// Calculate the next interval.                          // doc comment: bad — fragment, not a sentence
func nextInterval() time.Duration { … }
```

### Smells

- Doc comment in fragment form (`// gets the user`) — must be a sentence (`// User returns the user with the given ID.`).
- End-of-line struct-field comment that pretends to be a sentence (`// The address.`) — drop the period and capital, or expand into a real sentence above the field.
- Doc comment that ends without a period.

---

## #examples — Examples

> Packages should clearly document their intended usage. Try to provide a runnable example; examples show up in Godoc. Runnable examples belong in the test file, not the production source file.

> If it isn't feasible to provide a runnable example, example code can be provided within code comments. As with other code and command-line snippets in comments, it should follow standard formatting conventions.

### Runnable example shape (`*_test.go`)

```go
// In package_test.go:
func ExamplePackage() {
    p := mypkg.New()
    fmt.Println(p.Hello())
    // Output: hello world
}
```

The `// Output:` line is what makes the example *runnable* (`go test` verifies it).

### Smells

- Runnable example placed in production source (`example_test.go`-style code in `foo.go`) — move to a `*_test.go`.
- Example without an `// Output:` block — fine if intentional, but `go test` won't verify it; consider adding output if deterministic.
- Example function not named with the `Example`, `ExampleType`, `ExampleType_method`, or `ExampleFunction_suffix` shape — Godoc won't surface it.
- A complex package with no examples at all and only fragmentary comments — the doc explicitly asks for "clearly document intended usage."

---

## #named-result-parameters — Named result parameters

> When naming parameters, consider how function signatures appear in Godoc. The name of the function itself and the type of the result parameters are often sufficiently clear.

```go
func (n *Node) Parent1() *Node
func (n *Node) Parent2() (*Node, error)
```

> If a function returns two or more parameters of the same type, adding names can be helpful.

```go
func (n *Node) Children() (left, right *Node, err error)
```

> If the caller must take action on particular result parameters, naming them can help suggest what the action is.

(The Decisions doc gives `WithTimeout` as the canonical example — naming the returned `cancel` makes the call site requirement obvious.)

> Don't use named result parameters when the names produce unnecessary repetition.

> Don't name result parameters in order to avoid declaring a variable inside the function.

> Naked returns are acceptable only in a small function.

> It is always acceptable to name a result parameter if its value must be changed in a deferred closure.

### Decision tree

| Function returns... | Name the results? |
|---|---|
| Single value with self-evident type | No (`Parent() *Node`) |
| Single value + error | No (`Parse() (*Node, error)`) |
| Two+ values of the **same type** | Yes (`Children() (left, right *Node, err error)`) |
| A value the caller must act on (cancel func, cleanup func) | Yes — name suggests the action |
| Repeated names just to mirror parameters | No — that's the "unnecessary repetition" anti-pattern |
| To avoid a `var` declaration in the body | No — declare the variable |
| Used in a deferred closure to mutate the return | Yes |

### Smells

- Three-letter named returns that just echo the type (`func Foo() (n int, b bool, s string)` with no semantic role).
- Naked `return` in a function longer than ~10 lines — fine in tiny functions; opaque in larger ones.
- Named results paired with a body-level `:=` that re-introduces the same name — pick one.
- Single-return function with named result for no documentation reason (`func Count() (n int)` adds nothing over `func Count() int`).

---

## #package-comments — Package comments

> Package comments must appear immediately above the package clause with no blank line between the comment and the package name.

```go
// Package math provides basic constants and mathematical functions.
package math
```

> There must be a single package comment per package. If a package is composed of multiple files, exactly one of the files should have a package comment.

> Comments for main packages have a slightly different form, where the name of the go_binary rule in the BUILD file takes the place of the package name.

### Acceptable forms for `main` packages

The doc lists multiple acceptable phrasings (Google-internal context — `go_binary` rule names):

```go
// Binary seed_generator generates seed data for tests.
package main

// Command seed_generator generates seed data for tests.
package main

// Program seed_generator generates seed data for tests.
package main

// The seed_generator command generates seed data for tests.
package main
```

For non-Google projects: `// Command <name>` or `// <Name>` style is conventional.

### Smells

- No package comment at all in a non-trivial package.
- Package comment with a blank line between it and `package foo`.
- Package comment in **every** file of a multi-file package — should be in exactly one (typically `doc.go` or the most-foundational file).
- Package comment that doesn't begin with `Package <name>` (or `Binary/Command/Program <name>` for `main`).
- Stale package comment that references types/features no longer in the package.

---

## Consolidated review checklist (all 6 rules)

- [ ] Comment line length is consistent within the file (80 or 100 are common; not a hard rule).
- [ ] Every exported top-level symbol has a doc comment that **begins with the symbol's name** and is a full sentence.
- [ ] Unexported symbols with non-obvious behaviour also have doc comments.
- [ ] No blank line between a doc comment and the symbol it describes.
- [ ] Doc comments are full sentences (capitalized, period); end-of-line struct-field comments may be fragments.
- [ ] Packages provide runnable `Example*` functions in `*_test.go` for non-trivial APIs.
- [ ] Named result parameters appear only when (a) two-of-same-type, or (b) caller must act on the result, or (c) deferred closure must mutate.
- [ ] Naked returns confined to small functions.
- [ ] Each package has exactly one `// Package <name> …` comment in exactly one file, immediately above `package <name>` with no blank line.
- [ ] `main` packages use the `// Binary/Command/Program <name>` form.

## Suggested finding phrasing

- "Exported `func ParseURL(...)` has no doc comment — Decisions §doc-comments: required for all exported top-level names; comment must begin with `// ParseURL`."
- "`// gets the user` doc comment on `User()` — Decisions §comment-sentences + §doc-comments: doc comments must be full sentences beginning with the symbol name (`// User returns …`)."
- "Blank line between `// Server is …` and `type Server struct` — Decisions §doc-comments: no blank line between comment and symbol."
- "Function `func Foo() (n int, b bool, s string)` returns three different types and names them — Decisions §named-result-parameters: drop the names; types are sufficient for the Godoc reader."
- "Naked `return` at line 87 of a 60-line function — Decisions §named-result-parameters: naked returns acceptable only in small functions."
- "Package `myteampb` has no `// Package myteampb …` comment in any file — Decisions §package-comments: required, exactly once."
- "`// Package math provides …` followed by a blank line, then `package math` — Decisions §package-comments: no blank line between comment and package clause."
- "200-column comment line in the middle of a file where every other comment wraps at 80 — Decisions §comment-line-length: aim for consistency within the file."

## Sources

- <https://google.github.io/styleguide/go/decisions#commentary> — Google Go Style Decisions, Commentary section (the canonical text quoted above)
- <https://pkg.go.dev/go/doc> — Godoc rendering rules (referenced by `#doc-comments` for how comments surface)
- <https://pkg.go.dev/testing#hdr-Examples> — Go `testing` package on `Example*` functions (referenced by `#examples`)
