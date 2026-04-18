---
section: "Package size"
number: 2
category: philosophical
rules:
  - package-size
sources:
  - https://google.github.io/styleguide/go/best-practices#package-size
  - https://go.dev/blog/package-names
  - https://go.dev/blog/organizing-go-code
---

# Section 02 — Package size (Best Practices)

## What this section covers

Guidance on how big a Go package should be, when to split, when to combine, and how to organize source files within a package. This is **philosophical**: there's no mechanical rule a subagent can apply. The section informs the *judgment* a reviewer brings when commenting on package shape.

> [!IMPORTANT]
> Package size is rarely a single-PR finding — it's a structural concern that surfaces when a package starts growing or when a new package is created. Use this section's lens during architecture review, not on every per-line review.

## When to combine multiple types into one package

> If client code is likely to need two values of different type to interact with each other, it may be convenient for the user to have them in the same package.

> Code within a package can access unexported identifiers in the package. If you have a few related types whose implementation is tightly coupled, placing them in the same package lets you achieve this coupling without polluting the public API with these details.

The Best Practices test for "should these two types share a package?":

> A good test for this coupling is to imagine a hypothetical user of two packages, where the packages cover closely related topics: if the user must import both packages in order to use either in any meaningful way, combining them together is usually the right thing to do.

> The standard library generally demonstrates this kind of scoping and layering well.

## When to split

> All of that being said, putting your entire project in a single package would likely make that package too large.

> When something is conceptually distinct, giving it its own small package can make it easier to use.

> The short name of the package as known to clients together with the exported type name work together to make a meaningful identifier: e.g. `bytes.Buffer`, `ring.New`.

## File size within a package

> Go style is flexible about file size, because maintainers can move code within a package from one file to another without affecting callers.

> But as a general guideline: it is usually not a good idea to have a single file with many thousands of lines in it, or having many tiny files.

> There is no "one type, one file" convention as in some other languages.

> As a rule of thumb, files should be focused enough that a maintainer can tell which file contains something, and the files should be small enough that it will be easy to find once there.

> The standard library often splits large packages to several source files, grouping related code by file.

> The source for package `bytes` is a good example.

### `doc.go` convention

> Packages with long package documentation may choose to dedicate one file called `doc.go` that has the package documentation, a package declaration, and nothing else, but this is not required.

### Bazel / monorepo note

> Within the Google codebase and in projects using Bazel, directory layout for Go code is different than it is in open source Go projects: you can have multiple `go_library` targets in a single directory.

> A good reason to give each package its own directory is if you expect to open source your project in the future.

## Reference examples (from the doc)

| Size | Stdlib example | File layout |
|---|---|---|
| **Small** | `encoding/csv` | `reader.go` + `writer.go` (responsibility-split) |
| **Small** | `expvar` | All in one `expvar.go` |
| **Moderate** | `flag` | All in one `flag.go` |
| **Large** | `net/http` | `client.go`, `server.go`, `cookie.go`, … (split by concept) |
| **Large** | `os` | `exec.go`, `file.go`, `tempfile.go`, … (split by concept) |

## Why this isn't a per-line review rule

A subagent cannot mechanically detect "this package is the wrong size" — the answer depends on the package's domain, the imports of its callers, and the team's roadmap. So #02 is a **lens**, not a finding generator. Use it as background context when:

- Reviewing a new package addition — does the new package earn its own directory, or should it merge into a sibling?
- Reviewing a long file (>1000 lines) — is the file overdue for a split, or is the cohesion still there?
- Reviewing many tiny files (5+ files of <50 lines each) — could related fragments combine?
- Reviewing a `util`-shaped package (also see `#util-packages` in section 01) — should it split into domain-named packages?
- Reviewing imports — if two sibling packages are *always* imported together, they may need to be combined per the doc's "test for coupling."

## How reviewers should use it

When package size feels off, frame the comment as a question, not a directive:

- "This file has grown to ~1.8k lines covering both client and server logic — would splitting `client.go` / `server.go` (like `net/http` does) make this easier to navigate?"
- "Three new packages here (`foo/parser`, `foo/validator`, `foo/model`) all import each other. Per Best Practices §package-size: if a user can't meaningfully use any one without the others, consider merging into a single `foo` package."
- "New `internal/util` package — Best Practices §util-packages discourages this; can the helpers move into a domain-named package (`foo/parsing`, `foo/timeutil`) instead?"
- "Package documentation here is very long; consider extracting it into a dedicated `doc.go` file (Best Practices: optional but supported convention)."

## When NOT to apply

- **Stable, mature packages** with established structure — don't propose churn for cosmetic reasons.
- **Generated-code directories** — file count and size are governed by the generator.
- **Vendored / third-party code** — leave alone; structure belongs to upstream.
- **Mid-migration packages** — the awkward shape may be intentional during a refactor.
- **Bazel projects with multiple `go_library`s per directory** — applying open-source one-package-per-dir advice doesn't fit.

## Sources

- <https://google.github.io/styleguide/go/best-practices#package-size> — Google Go Best Practices, Package size section (the canonical text quoted above)
- <https://go.dev/blog/package-names> — *Package names* (referenced as the foundational reading)
- <https://go.dev/blog/organizing-go-code> — *Organizing Go Code* (Go blog)
- <https://go.dev/talks/2014/organizeio.slide> — *Organizing Go Code* (presentation)
- <https://google.github.io/styleguide/go/decisions#package-comments> — `decisions-review:02-commentary.md#package-comments` (referenced for the `doc.go` convention)
