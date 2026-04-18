---
section: "Imports"
number: 3
category: actionable
rules:
  - import-renaming
  - import-grouping
  - import-blank
  - import-dot
sources:
  - https://google.github.io/styleguide/go/decisions#imports
---

# Section 03 — Imports

## What this section covers

Four concrete decisions on how `import (...)` blocks are written: when (and how) to rename, how to group, when blank imports (`_`) are allowed, and the strict ban on dot imports (`.`).

> [!IMPORTANT]
> The mechanical part of import grouping is enforced by `goimports` (and reinforced by `gofumpt`). Most repos with one of those tools wired into CI / pre-commit will satisfy `#import-grouping` automatically — but the *renaming* and *blank/dot* decisions are content-level and need a real review.

## #import-renaming — Import renaming

> Package imports shouldn't normally be renamed, but there are cases where they must be renamed or where a rename improves readability.

> Local names for imported packages must follow the guidance around package naming, including the prohibition on the use of underscores and capital letters.

When renaming **is** required or recommended:

1. **Name collision** between two imports — must rename one.
2. **Generated protobuf packages** — must be renamed to remove underscores; local names must have a `pb` suffix:

   ```go
   import (
       foosvcpb "path/to/package/foo_service_go_proto"
   )
   ```

3. **Uninformative original name** (`util`, `v1`, etc.) — rename to something meaningful:

   ```go
   import (
       core "github.com/kubernetes/api/core/v1"
       meta "github.com/kubernetes/apimachinery/pkg/apis/meta/v1beta1"
   )
   ```

4. **Collision with a common local variable name** (`url`, `ssh`, `time`, `name`, `count`) — preferred convention is the `pkg` suffix (`urlpkg`, `sshpkg`).

### Smells

- Renamed import where no collision, no uninformative name, and not a protobuf — usually a personal-preference rename, drop it.
- Protobuf import without a `pb` suffix (`import foo "path/to/foo_go_proto"` instead of `foopb`).
- Protobuf import that still has underscores in the local name.
- Renamed import using underscores or capital letters in the local name (`my_pkg "..."` or `MyPkg "..."`).
- Local name that itself collides with a common variable name (renamed `url` to `urlpkg` is good; renamed to plain `url` defeats the point).

---

## #import-grouping — Import grouping

> Imports should be organized into the following groups, in order:
> 1. Standard library packages
> 2. Other (project and vendored) packages
> 3. Protocol Buffer imports (e.g., `fpb "path/to/foo_go_proto"`)
> 4. Import for side-effects (e.g., `_ "path/to/package"`)

Each group is separated by a blank line. Within a group, imports are alphabetical (`gofmt`/`goimports` enforces this).

### Canonical example

```go
package main

import (
    "fmt"
    "hash/adler32"
    "os"

    "github.com/dsnet/compress/flate"
    "golang.org/x/text/encoding"
    "google.golang.org/protobuf/proto"

    foopb "myproj/foo/proto/proto"

    _ "myproj/rpc/protocols/dial"
    _ "myproj/security/auth/authhooks"
)
```

Four blank-line-separated groups: stdlib → external → protobuf → side-effect.

### Smells

- Mixed stdlib and external in the same group (no blank line between `"fmt"` and `"github.com/..."`).
- Protobuf imports interleaved with non-pb imports instead of being a separate group.
- Side-effect (`_`) imports placed above the named imports.
- A 5th custom group (e.g., "internal" pulled out of "other") — not in the canonical 4; flag unless the project explicitly documents the deviation.
- Imports not alphabetical within a group — `goimports` should fix this; if not, the formatter isn't running.

---

## #import-blank — Import "blank" (`import _`)

> Packages that are imported only for their side effects (using the syntax `import _ "package"`) may only be imported in a main package, or in tests that require them.

Examples the doc names: `"time/tzdata"`, `"image/jpeg"` in image processing code.

> Avoid blank imports in library packages, even if the library indirectly depends on them.

> Constraining side-effect imports to the main package helps control dependencies, and makes it possible to write tests that rely on a different import without conflict or wasted build costs.

### Two explicit exceptions

1. **`nogo` bypass** — you may use a blank import to bypass the check for disallowed imports in the `nogo` static checker.
2. **`embed` package** — you may use a blank import of the `embed` package in a source file which uses the `//go:embed` compiler directive.

### Tip from the doc

> If you create a library package that indirectly depends on a side-effect import in production, document the intended usage.

That is: don't smuggle the side-effect into a library — surface it for the binary to pick up.

### Smells

- `_ "..."` in a non-`main` library package (where it's not `embed` or `nogo`).
- A library that needs a side-effect to function but hides the requirement instead of documenting it for binary authors.
- Blank import of `embed` without an actual `//go:embed` directive in the file.
- Side-effect imports listed in any group other than the dedicated 4th group (see `#import-grouping`).

---

## #import-dot — Import "dot" (`import .`)

> The `import .` form is a language feature that allows bringing identifiers exported from another package to the current package without qualification.

> Do not use this feature in the Google codebase; it makes it harder to tell where the functionality is coming from.

### Bad

```go
package foo_test

import (
    "bar/testutil" // also imports "foo"
    . "foo"
)

var myThing = Bar() // Bar defined in package foo; no qualification needed.
```

### Good

```go
package foo_test

import (
    "bar/testutil" // also imports "foo"
    "foo"
)

var myThing = foo.Bar()
```

### Smells

- Any `. "..."` import — flag it.
- Justification "but the test is verbose otherwise" — Decisions is absolute on this; Google codebase rule. (Some open-source projects do use dot imports, e.g. Ginkgo's BDD style — that's a project-level convention divergence, not a Google-style allowance.)

---

## Consolidated review checklist (all 4 rules)

- [ ] Imports are renamed only for: name collisions, protobuf packages (`pb` suffix), uninformative original names, or local-variable name conflicts (`pkg` suffix).
- [ ] Generated protobuf imports use the `pb` suffix and have no underscores in the local name.
- [ ] All renamed local names follow Go package naming (lowercase, no underscores).
- [ ] Imports are organized into the canonical 4 groups (stdlib / other / pb / side-effect), each separated by a blank line.
- [ ] Within each group, imports are alphabetical.
- [ ] Side-effect imports (`_ "..."`) appear only in `main` or test packages — except `embed` (with `//go:embed`) or `nogo` bypass.
- [ ] Library packages do not blank-import side effects; side-effect requirements are documented for binary authors instead.
- [ ] No `import .` (dot imports) anywhere.

## Suggested finding phrasing

- "`import . \"foo\"` in `bar_test.go` — Decisions §import-dot: dot imports are banned; use `\"foo\"` and qualify call sites with `foo.`."
- "Protobuf import `import foo \"path/to/foo_go_proto\"` lacks the `pb` suffix — Decisions §import-renaming: use `foopb`."
- "Side-effect import `_ \"github.com/x/driver\"` in non-`main` library package `internal/db` — Decisions §import-blank: blank imports allowed only in `main` or tests; move to the binary's `main.go` (or `embed`/`nogo` exceptions)."
- "Imports not grouped: stdlib `\"fmt\"` adjacent to external `\"github.com/...\"` with no blank line — Decisions §import-grouping: 4 groups (stdlib / other / pb / side-effect) separated by blank lines."
- "Import renamed without a stated reason (`alias \"github.com/.../foo\"` where `foo` would not collide and is informative) — Decisions §import-renaming: drop the rename."

## Sources

- <https://google.github.io/styleguide/go/decisions#imports> — Google Go Style Decisions, Imports section (the canonical text quoted above)
- <https://pkg.go.dev/golang.org/x/tools/cmd/goimports> — `goimports` (the formatter that enforces grouping/alphabetization)
- <https://pkg.go.dev/embed> — Go `embed` package (one of the two blank-import exceptions)
