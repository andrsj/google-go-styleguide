---
section: "Imports"
number: 3
category: actionable
rules:
  - protocol-buffer-messages-and-stubs
  - import-ordering
sources:
  - https://google.github.io/styleguide/go/best-practices#imports
  - https://google.github.io/styleguide/go/decisions#import-grouping
---

# Section 03 — Imports (Best Practices)

## What this section covers

Two recommendations on imports — the naming convention for protobuf and gRPC import aliases (more detail than `decisions-review:03-imports.md#import-renaming`), and a deferral to the Decisions document for ordering.

> [!IMPORTANT]
> The Decisions doc covers the structural rules (4 groups, blank-line separated, no dot imports, when to rename). This Best Practices section adds the nuance for **proto/gRPC alias naming specifically** — prefer whole-word descriptive names over the older "xpb" / "pb" shorthand.

## Protocol Buffer Messages and Stubs

> Proto library imports are treated differently than standard Go imports due to their cross-language nature.

### Suffix convention

> The convention for renamed proto imports are based on the rule that generated the package:
>
> - The `pb` suffix is generally used for `go_proto_library` rules.
> - The `grpc` suffix is generally used for `go_grpc_library` rules.

### Naming guidance

> Often a single word describing the package is used:

```go
// Good:
import (
    foopb   "path/to/package/foo_service_go_proto"
    foogrpc "path/to/package/foo_service_go_grpc"
)
```

> Follow the style guidance for package names. Prefer whole words. Short names are good, but avoid ambiguity.

When unsure:

> When in doubt, use the proto package name up to `_go` with a `pb` suffix:

```go
// Good:
import (
    pushqueueservicepb "path/to/package/push_queue_service_go_proto"
)
```

### Important note on legacy short names

> **Note:** Previous guidance encouraged very short names such as `xpb` or even just `pb`. New code should prefer more descriptive names. Existing code which uses short names should not be used as an example, but does not need to be changed.

So the rule has two tiers:

| Codebase context | Acceptable proto import alias |
|---|---|
| **New code** | Whole-word descriptive (`foopb`, `pushqueueservicepb`, `foogrpc`) |
| **Existing code with `xpb` / `pb` / `mpb`** | Don't churn it — leave alone unless you're already touching the file for other reasons |
| **Modifying file with old-style naming + adding new proto import** | Mixed: add new ones with descriptive names; consider renaming old aliases if the diff stays small. Otherwise local consistency wins (see `guide-review:10-local-consistency.md`). |

### Smells

- New-code proto import using `pb` or `xpb` alone — should be a descriptive whole-word name.
- gRPC import without a `grpc` suffix — `import foo "..._go_grpc"` instead of `foogrpc`.
- Proto import without **any** suffix (`import foo "path/foo_go_proto"`) — should be `foopb`.
- Inconsistent suffix style within a file (one import uses `pb`, another uses no suffix).
- Proto import alias that includes the `pb_` prefix (`pb_foo` instead of `foopb`) — suffix, not prefix.
- A single-letter alias for a proto (`p "..._go_proto"`) — opaque even if it works.

### Rare-but-acceptable exceptions

- Test code with many proto imports that's clearly local — short suffixes are tolerated for readability.
- Generated code that imports its own proto stubs — the generator's convention wins.

---

## Import ordering

> See the [Go Style Decisions: Import grouping](https://google.github.io/styleguide/go/decisions#import-grouping).

This section contains no additional content — Best Practices defers entirely to the Decisions doc on import grouping. The 4-group convention (stdlib / other / pb / side-effect) and alphabetical-within-group rule live in `decisions-review:03-imports.md#import-grouping`. Reviews of import ordering should reference that section directly, not this one.

### What you should already know from `decisions-review`

- Imports go into 4 groups separated by blank lines.
- Within a group, imports are alphabetical (`goimports` enforces this).
- Side-effect (`_`) imports go in the dedicated 4th group.
- `gofumpt` further normalizes some edge cases.

### Smells (reuses `decisions-review:03-imports.md`)

- Mixed groups (no blank line separating stdlib from external).
- Protobuf imports interleaved with non-pb imports.
- Side-effect imports above named imports.
- Imports not alphabetical within a group.

If you find any of these, reference `decisions-review:03-imports.md#import-grouping` in the finding, not this section.

---

## Consolidated review checklist

- [ ] New proto imports use whole-word descriptive aliases with `pb` suffix (`foopb`, not `pb` or `xpb`).
- [ ] New gRPC imports use whole-word descriptive aliases with `grpc` suffix (`foogrpc`).
- [ ] Suffix is at the end of the alias, not the start (`foopb`, not `pb_foo`).
- [ ] Existing code using legacy `xpb`/`pb` short forms is not churned just for renaming — local consistency rules.
- [ ] Import ordering follows `decisions-review:03-imports.md#import-grouping` (4 groups, blank-line separated, alphabetical).

## Suggested finding phrasing (advisory)

- "New proto import `import pb \"path/foo_service_go_proto\"` — Best Practices §protocol-buffer-messages-and-stubs: use a descriptive whole-word alias for new code (`foopb`, `fooservicepb`, etc.). Legacy `pb` alone is no longer recommended."
- "gRPC import `import foo \"path/foo_service_go_grpc\"` lacks the `grpc` suffix — Best Practices: use `foogrpc`."
- "Proto import alias `pb_foo` — Best Practices: suffix goes at the end (`foopb`), not the start."
- "Existing `xpb` imports in a file you're modifying — Best Practices says these don't *need* to be changed; only rename if the rest of the change scope already touches these lines."
- "Imports not properly grouped — see `decisions-review:03-imports.md#import-grouping` for the 4-group rule."

## Sources

- <https://google.github.io/styleguide/go/best-practices#imports> — Google Go Best Practices, Imports section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/decisions#import-grouping> — Style Decisions, Import grouping (covered by sibling skill `decisions-review:03-imports.md`)
- <https://google.github.io/styleguide/go/decisions#package-names> — Style Decisions, Package names (referenced by the proto-naming guidance)
- <https://pkg.go.dev/golang.org/x/tools/cmd/goimports> — `goimports` (enforces grouping/alphabetization)
