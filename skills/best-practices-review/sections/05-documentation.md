---
section: "Documentation"
number: 5
category: actionable
rules:
  - conventions (parameters-and-configuration, contexts, concurrency, cleanup, errors)
  - preview
  - godoc-formatting
  - signal-boost
sources:
  - https://google.github.io/styleguide/go/best-practices#documentation
---

# Section 05 — Documentation (Best Practices)

## What this section covers

Pragmatic guidance on *what* to document and *how* to format it for Godoc — beyond the structural rules in `decisions-review:02-commentary.md` (which covers when doc comments are required, sentence shape, package comments, etc.). Best Practices adds: which parameters / contexts / concurrency expectations / cleanup / errors are worth calling out, how to format for Godoc rendering, and the *signal boost* pattern for lines that look common but aren't.

> [!IMPORTANT]
> The structural rules (every exported symbol gets a doc comment, comment begins with the symbol name, no blank line between comment and symbol, etc.) live in `decisions-review:02-commentary.md`. This section is about *content quality* once those structural rules are satisfied.

## Conventions

### Parameters and configuration

> Not every parameter must be enumerated in the documentation. This applies to: function and method parameters, struct fields, APIs for options.

> Document the error-prone or non-obvious fields and parameters by saying *why* they are interesting.

#### Bad — restating without adding info

```go
// Sprintf formats according to a format specifier and returns the resulting
// string.
//
// format is the format, and data is the interpolation data.
func Sprintf(format string, data ...any) string
```

The `format is the format, and data is the interpolation data` part adds nothing the reader can't infer from the parameter names.

#### Good — non-obvious behavior surfaced

```go
// Sprintf formats according to a format specifier and returns the resulting
// string.
//
// The provided data is used to interpolate the format string. If the data does
// not match the expected format verbs or the amount of data does not satisfy
// the format specification, the function will inline warnings about formatting
// errors into the output string as described by the Format errors section
// above.
func Sprintf(format string, data ...any) string
```

> Consider your likely audience in choosing what to document and at what depth.

### Contexts

> It is implied that the cancellation of a context argument interrupts the function it is provided to. If the function can return an error, conventionally it is `ctx.Err()`.

> This fact does not need to be restated.

#### Bad — restating the convention

```go
// Run executes the worker's run loop.
//
// The method will process work until the context is cancelled and accordingly
// returns an error.
func (Worker) Run(ctx context.Context) error
```

#### Good — concise

```go
// Run executes the worker's run loop.
func (Worker) Run(ctx context.Context) error
```

But document context behavior **when it differs from the convention** or has special expectations.

#### Good — non-default cancellation behavior

```go
// Run executes the worker's run loop.
//
// If the context is cancelled, Run returns a nil error.
func (Worker) Run(ctx context.Context) error
```

#### Good — multiple shutdown mechanisms

```go
// Run executes the worker's run loop.
//
// Run processes work until the context is cancelled or Stop is called.
// Context cancellation is handled asynchronously internally: run may return
// before all work has stopped. The Stop method is synchronous and waits
// until all operations from the run loop finish. Use Stop for graceful
// shutdown.
func (Worker) Run(ctx context.Context) error

func (Worker) Stop()
```

#### Good — special expectations

```go
// NewReceiver starts receiving messages sent to the specified queue.
// The context should not have a deadline.
func NewReceiver(ctx context.Context) *Receiver

// Principal returns a human-readable name of the party who made the call.
// The context must have a value attached to it from security.NewContext.
func Principal(ctx context.Context) (name string, ok bool)
```

### Concurrency

> Go users assume that conceptually read-only operations are safe for concurrent use and do not require extra synchronization.

#### Don't restate the obvious read-only convention

```go
// Bad-leaning (unnecessary):
// Len returns the number of bytes of the unread portion of the buffer;
// b.Len() == len(b.Bytes()).
//
// It is safe to be called concurrently by multiple goroutines.
func (*Buffer) Len() int
```

> Mutating operations, however, are not assumed to be safe for concurrent use and require the user to consider synchronization.

Document concurrency when behavior is unclear or the API guarantees thread safety:

#### Good — flag that mutating op is NOT safe

```go
package lrucache

// Lookup returns the data associated with the key from the cache.
//
// This operation is not safe for concurrent use.
func (*Cache) Lookup(key string) (data []byte, ok bool)
```

#### Good — flag that the API IS safe (when otherwise non-obvious)

```go
package fortune_go_proto

// NewFortuneTellerClient returns an *rpc.Client for the FortuneTeller service.
// It is safe for simultaneous use by multiple goroutines.
func NewFortuneTellerClient(cc *rpc.ClientConn) *FortuneTellerClient
```

#### Good — set expectations for user-implemented types

```go
package health

// A Watcher reports the health of some entity (usually a backend service).
//
// Watcher methods are safe for simultaneous use by multiple goroutines.
type Watcher interface {
    Watch(changed chan<- bool) (unwatch func())
    Health() error
}
```

### Cleanup

> Document any explicit cleanup requirements that the API has.

#### Good — call out caller's responsibility

```go
// NewTicker returns a new Ticker containing a channel that will send the
// current time on the channel after each tick.
//
// Call Stop to release the Ticker's associated resources when done.
func NewTicker(d Duration) *Ticker

func (*Ticker) Stop()
```

#### Good — show the cleanup pattern

```go
// Get issues a GET to the specified URL.
//
// When err is nil, resp always contains a non-nil resp.Body.
// Caller should close resp.Body when done reading from it.
//
//    resp, err := http.Get("http://example.com/")
//    if err != nil {
//        // handle error
//    }
//    defer resp.Body.Close()
//    body, err := io.ReadAll(resp.Body)
func (c *Client) Get(url string) (resp *Response, err error)
```

### Errors

> Document significant error sentinel values or error types that your functions return to callers so that callers can anticipate what types of conditions they can handle in their code.

#### Good — name the sentinel

```go
package os

// Read reads up to len(b) bytes from the File and stores them in b. It returns
// the number of bytes read and any error encountered.
//
// At end of file, Read returns 0, io.EOF.
func (*File) Read(b []byte) (n int, err error)
```

#### Good — note pointer-vs-value of returned error type

```go
package os

type PathError struct {
    Op   string
    Path string
    Err  error
}

// Chdir changes the current working directory to the named directory.
//
// If there is an error, it will be of type *PathError.
func Chdir(dir string) error
```

> Documenting whether the values returned are pointer receivers enables callers to correctly compare the errors using `errors.Is`, `errors.As`, and package `cmp`.

> Document overall error conventions in the package's documentation when the behavior is applicable to most errors found in the package.

```go
// Package os provides a platform-independent interface to operating system
// functionality.
//
// Often, more information is available within the error. For example, if a
// call that takes a file name fails, such as Open or Stat, the error will
// include the failing file name when printed and will be of type *PathError,
// which may be unpacked for more information.
package os
```

### Smells (across all `Conventions` sub-rules)

- Doc comment that re-states what parameter names already convey ("`format` is the format string").
- Doc comment that restates the default context-cancellation convention ("returns an error when ctx is cancelled").
- Concurrency note on a clearly read-only accessor ("safe for concurrent use" on a `Len()` method).
- Mutating method missing concurrency note (no "not safe for concurrent use" annotation on a `Cache.Set`).
- API that needs cleanup but doesn't document it (`NewWatcher` that returns a goroutine handle without telling the caller to `Close()`).
- Functions returning sentinel errors / typed errors without documenting which (e.g., `Open` returns `*os.PathError` but the doc comment doesn't say so).
- Package-level error convention applied across many functions but not stated in the package comment.

---

## #preview — Preview

> Go features a [documentation server](https://pkg.go.dev/golang.org/x/pkgsite/cmd/pkgsite). It is recommended to preview the documentation your code produces both before and during the code review process.

Practical command:

```sh
go install golang.org/x/pkgsite/cmd/pkgsite@latest
pkgsite -open .   # in your project root
```

This is *advice for the author* during writing/review, not a per-line review rule. Reviewers can ask "did you preview this in pkgsite?" if doc rendering looks suspicious.

### Smells

- Doc comments that contain markdown that doesn't render as expected in Godoc (e.g., `**bold**` — Godoc doesn't render markdown bold; treats it as literal asterisks).
- Embedded URLs without using Godoc's auto-link convention (Godoc auto-links `https://...`).
- Doc comments with inline backticks expecting markdown code rendering — Godoc treats them literally.

---

## #godoc-formatting — Godoc formatting

> Godoc provides some specific syntax to format documentation.

The four conventions:

| Element | How to format in a doc comment |
|---|---|
| **Paragraph break** | A blank comment line (`//` on its own) between two paragraphs |
| **Runnable examples** | Defined as `ExampleX`, `ExampleType_Method`, etc. in `*_test.go`; Godoc attaches them to the doc |
| **Verbatim block** (code, ASCII art) | Indent the lines by an additional 2 spaces |
| **Header** | A single line starting with a capital letter, no punctuation except `(),`, followed by another paragraph |

### Example

```go
// Process applies the rules to the input.
//
// The implementation runs three phases:
//
// Validation
//
// First, the input is validated against the schema...
//
// Transformation
//
// Then, transformations are applied in order...
//
// Example usage:
//
//   p := NewProcessor()
//   out, err := p.Process(in)
//   if err != nil {
//       log.Fatal(err)
//   }
func Process(in *Input) (*Output, error)
```

### Smells

- Two paragraphs joined without a blank `//` line — Godoc renders as one paragraph.
- Code block in doc comment with single-space or no indentation — won't render verbatim.
- "Header" line with trailing punctuation (`Validation:`) — Godoc won't recognize it as a header.
- Markdown syntax (`**bold**`, `_italic_`, `[link](url)`) — treated as literal text by Godoc.
- Indented code in a doc comment that uses tabs (or non-2-space indent) — may not render reliably.

(See also `decisions-review:02-commentary.md#examples` for the runnable-example convention.)

---

## #signal-boost — Signal boosting

> Sometimes a line of code looks like something common, but actually isn't. One of the best examples of this is an `err == nil` check (since `err != nil` is much more common).

> You can instead "boost" the signal of the conditional by adding a comment.

The pattern:

```go
// Good:
val, err := compute()
// We expect this to succeed; only emit on success.
if err == nil {
    publish(val)
}
```

The comment draws the reader's eye to a deliberately-unusual conditional that would otherwise blend in with surrounding `if err != nil` checks and be misread.

### Other classic signal-boost moments

- `if err == nil { ... }` — opposite of the common pattern.
- `if !ok { ... }` after a map lookup where most local code uses `if ok { ... }`.
- `defer recover()` in a place a panic-recovery isn't expected.
- `_ = expensiveCleanup()` — explicit ignore of an error a reader might expect to be checked.
- Math that *looks* like a typo but isn't (`for i := n - 1; i >= 0; i--`).
- Loops that mutate a slice while iterating (when intentional).
- A `select` with only one `case` (semantically just a receive, but might be deliberate for cancellability).

The Best Practices doc presents this as a *Conventions for clarity* tool — comments aren't decoration, they're a deliberate signal-boost when the code's shape would mislead a quick reader.

### Smells (anti-patterns this rule responds to)

- `if err == nil { ... }` with no comment — easy to misread as `if err != nil`.
- Inverted condition for genuinely unusual logic, with no clarifying comment.
- A line with subtle but important semantic difference from a common idiom (e.g., `select { case <-ctx.Done(): }`) without a comment explaining why.
- Comments that explain *what* the code does (which is obvious) instead of *why* this case is unusual.

---

## Consolidated review checklist (4 sub-rules + sub-sub-rules)

### Conventions — content quality

- [ ] Doc comments add information beyond what parameter/field names already convey.
- [ ] Context behavior is documented only when non-default (no restating "ctx cancellation interrupts").
- [ ] Concurrency safety documented when non-obvious (mutating ops flagged "not safe"; thread-safe APIs flagged so).
- [ ] Cleanup requirements (`Stop()`, `Close()`, `Cancel()`) explicitly documented when caller-responsible.
- [ ] Significant sentinel errors and typed errors named in the doc; pointer-vs-value of typed errors specified.
- [ ] Package-wide error conventions (e.g., "all errors are `*PathError`") in the package comment.

### Preview & format

- [ ] Author has previewed the rendered doc in `pkgsite` for non-trivial new symbols (advisory).
- [ ] Paragraphs separated by blank `//` lines.
- [ ] Verbatim blocks indented by 2 spaces (not tabs, not markdown fences).
- [ ] Headers are bare capitalized lines with no trailing punctuation.
- [ ] No markdown syntax (`**bold**`, etc.) in doc comments.

### Signal boosting

- [ ] Deliberately-unusual lines (`err == nil`, inverted conditions, intentional mutation-during-iter) have a clarifying comment that explains *why*.

## Suggested finding phrasing (advisory)

- "Doc comment for `Sprintf` says \"format is the format string\" — Best Practices §parameters-and-configuration: the parameter name already conveys this; document the *why* (e.g., what happens with mismatched verbs)."
- "Doc comment says \"returns an error when ctx is cancelled\" — Best Practices §contexts: that's the default convention, no need to restate. Drop the line unless cancellation behaves non-default."
- "`func (*Cache) Set(...)` has no concurrency note, sibling `Get` does — Best Practices §concurrency: mutating ops should be flagged either way (\"not safe\" or, with synchronization, \"safe\")."
- "`NewWatcher` returns an object with goroutines but doesn't document a `Close()` requirement — Best Practices §cleanup: callers won't know to clean up."
- "`Open` returns `*PathError` but doc comment says only `error` — Best Practices §errors: name the type so callers can `errors.As`."
- "Package `os` documents `*PathError` convention in many functions individually but not in the package comment — Best Practices §errors: when a convention applies broadly, state it once in the package comment."
- "Doc comment uses `**bold**` markdown — Best Practices §godoc-formatting: Godoc treats this as literal asterisks."
- "Two-paragraph doc comment joined without a blank `//` line — Best Practices §godoc-formatting: Godoc will render as one paragraph."
- "`if err == nil { publish(val) }` with no comment — Best Practices §signal-boost: this inverts the common `err != nil` pattern; add a brief comment explaining why."

## Sources

- <https://google.github.io/styleguide/go/best-practices#documentation> — Google Go Best Practices, Documentation section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#commentary> — Decisions, Commentary (foundational rules — sibling skill `decisions-review:02-commentary.md`)
- <https://go.dev/doc/comment> — *Go Doc Comments* reference (Godoc syntax)
- <https://pkg.go.dev/golang.org/x/pkgsite/cmd/pkgsite> — `pkgsite` (preview tool referenced by `#preview`)
