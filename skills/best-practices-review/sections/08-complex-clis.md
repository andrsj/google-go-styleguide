---
section: "Complex command-line interfaces"
number: 8
category: actionable
rules:
  - complex-command-line-interfaces
sources:
  - https://google.github.io/styleguide/go/best-practices#complex-command-line-interfaces
  - https://pkg.go.dev/github.com/google/subcommands
  - https://pkg.go.dev/github.com/spf13/cobra
---

# Section 08 — Complex command-line interfaces (Best Practices)

## What this section covers

Library recommendations for Go binaries that need sub-commands (`kubectl create`, `kubectl run`, etc.). Best Practices recommends [`google/subcommands`](https://pkg.go.dev/github.com/google/subcommands) by default; allows [`spf13/cobra`](https://pkg.go.dev/github.com/spf13/cobra) when its extra features are required, with a specific context-handling pitfall to watch for.

> [!IMPORTANT]
> This section is mostly **library guidance**, not per-line review rules. Reviews against this section land at three points: (a) when a new binary picks a CLI library, (b) when the chosen library is `cobra` and context handling needs verification, (c) when subcommand structure / package boundaries need a sanity check.

## What it says

> Some programs wish to present users with a rich command-line interface that includes sub-commands.

> For example, `kubectl create`, `kubectl run`, and many other sub-commands are all provided by the program `kubectl`.

> There are at least the following libraries in common use for achieving this.

> If you don't have a preference or other considerations are equal, [subcommands](https://pkg.go.dev/github.com/google/subcommands) is recommended, since it is the simplest and is easy to use correctly.

> However, if you need different features that it doesn't provide, pick one of the other options.

### The two named libraries

| Library | Flag convention | Notes |
|---|---|---|
| **`google/subcommands`** | Go-style flags (`-foo`, `--foo`) | Simple, easy to use correctly. **Recommended default.** |
| **`spf13/cobra`** | `getopt` flags (`--foo`, `-f`) | Common outside Google. Many extra features. **Has context-handling pitfall.** |

### The cobra context pitfall

> cobra command functions should use `cmd.Context()` to obtain a context rather than creating their own root context with `context.Background`.

> Code that uses the subcommands package already receives the correct context as a function parameter.

So in cobra:

```go
// Bad — creates a fresh root context, loses cancellation/timeout from parent:
RunE: func(cmd *cobra.Command, args []string) error {
    ctx := context.Background()
    return doWork(ctx, args)
}

// Good — inherits the context cobra plumbed through (signals, parent cancel, etc.):
RunE: func(cmd *cobra.Command, args []string) error {
    ctx := cmd.Context()
    return doWork(ctx, args)
}
```

`subcommands` doesn't have this trap because the context is passed in directly to the command function.

### Subcommand package layout

> You are not required to place each subcommand in a separate package, and it is often not necessary to do so.

> Apply the same considerations about package boundaries as in any Go codebase.

So Go's general package-shape rules from `decisions-review` and `best-practices-review:02-package-size.md` apply here too — don't pre-fragment for cosmetic reasons; group cohesive subcommands together.

### Library/binary separation

> If your code can be used both as a library and as a binary, it is usually beneficial to separate the CLI code and the library, making the CLI just one more of its clients.

The recommended layout when a binary's logic could also be a library:

```
project/
├── cmd/
│   └── mybinary/
│       ├── main.go            # cobra/subcommands wiring only
│       └── cmd_*.go           # one file per subcommand
└── pkg/
    └── mybinary/
        ├── client.go          # the actual logic
        └── ...                # importable from other Go programs
```

The CLI in `cmd/` is just one consumer of the library in `pkg/`. Tests of the library don't need the CLI; consumers don't need the CLI.

## Decision tree

| You're building... | Use |
|---|---|
| New binary with sub-commands, no special needs | `google/subcommands` |
| New binary, must be familiar to non-Google contributors / open-source ecosystem | `spf13/cobra` (mind the context pitfall) |
| New binary needing features `subcommands` lacks (auto-completion, plugins, complex flag types) | `cobra` |
| Existing project on a different lib (e.g., `urfave/cli`) | Local consistency wins (see `guide-review:10-local-consistency.md`) — don't churn for taste |
| Tiny single-command binary | Neither library — bare `flag` is fine |

## Smells

- New binary using a third CLI library (`urfave/cli`, `kingpin`, `mitchellh/cli`) when `subcommands` would do — Best Practices doesn't ban these, but the recommendation is `subcommands` first, `cobra` second.
- `cobra` command function calling `context.Background()` instead of `cmd.Context()` — drops parent cancellation; this is the explicit warning in the doc.
- `cobra` command without a `RunE` (use `Run` only when no error path is possible) — `RunE` is the safer default.
- Each subcommand split into its own package "for cleanliness" with no actual coupling reason — premature decomposition; consolidate.
- A binary's main logic living in `package main` with no library extraction, even though the same code would be valuable as a library — `cmd/X/main.go` + `pkg/X/` split.
- `subcommands` `Execute` ignoring the returned status code — should be `os.Exit(int(subcommands.Execute(ctx)))` style.
- Cobra subcommand wired up but not hooked into the parent command (`rootCmd.AddCommand(subCmd)` missing).
- Cobra's `cobra.OnInitialize` callbacks doing heavy work that breaks `--help` (which still triggers init).

## When NOT to apply

- **Tiny single-command binaries** — `flag` is enough; no need for either library.
- **One-shot scripts / migrations** — overhead isn't worth it.
- **Existing binaries on a non-recommended library** — local consistency wins; don't migrate just to match this section.
- **Generated CLIs (e.g., from a proto or OpenAPI spec)** — generator's choice.
- **Wrappers around an external CLI** (e.g., a thin Go shell over `gcloud`) — usually no library at all.

## Suggested finding phrasing (advisory)

- "New binary using `urfave/cli` — Best Practices §complex-command-line-interfaces recommends `google/subcommands` (or `spf13/cobra` if you need its extra features). Migration not required if there's prior precedent in the repo."
- "Cobra command's `RunE` calls `context.Background()` — Best Practices §complex-command-line-interfaces warns specifically against this; use `cmd.Context()` to inherit parent cancellation and signal handling."
- "Each subcommand placed in its own package (`internal/cmds/foo`, `internal/cmds/bar`) with no cross-cutting requirement — Best Practices: \"You are not required to place each subcommand in a separate package.\" Consider consolidating."
- "Binary code in `cmd/mybin/main.go` reaches into business logic that other Go programs would benefit from — Best Practices: split CLI from library so `pkg/mybin` is reusable."
- "Cobra command uses `Run` (no error returned) but the body can fail — switch to `RunE` and return errors so cobra can surface them via `--help` / non-zero exit."

## Sources

- <https://google.github.io/styleguide/go/best-practices#complex-command-line-interfaces> — Google Go Best Practices, Complex CLIs section (the canonical text quoted above)
- <https://pkg.go.dev/github.com/google/subcommands> — `google/subcommands` (recommended default)
- <https://pkg.go.dev/github.com/spf13/cobra> — `spf13/cobra` (alternate; mind the context pitfall)
- <https://pkg.go.dev/flag> — Stdlib `flag` package (sufficient for single-command binaries)
- <https://google.github.io/styleguide/go/decisions#flags> — Decisions §flags (sibling skill `decisions-review:06-common-libraries.md` — flag-naming conventions still apply)
