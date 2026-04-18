---
guideline: "Formatting (gofmt)"
number: 6
category: tooling
sources:
  - https://google.github.io/styleguide/go/guide
  - https://pkg.go.dev/cmd/gofmt
  - https://pkg.go.dev/go/format#Source
  - https://github.com/mvdan/gofumpt
---

# #06 — Formatting (gofmt)

## What it requires

All Go source files must be formatted by `gofmt`, with no exceptions. The Style Guide states this in three sentences:

> All Go source files must conform to the format outputted by the `gofmt` tool.

> This format is enforced by a presubmit check in the Google codebase.

> Generated code should generally also be formatted (e.g., by using [`format.Source`](https://pkg.go.dev/go/format#Source)), as it is also browsable in Code Search.

That's the entire Style Guide section on formatting — short because the rule is short: **let `gofmt` decide**.

## Why it's a tooling rule, not a code-walk rule

Formatting is not something a reviewer should eyeball line-by-line. The check is:

1. The repo has `gofmt` (or `gofmt -d` / `gofmt -l`) wired up so that unformatted code can't land — typically in pre-commit hooks, CI, or both.
2. Generated code paths run through `format.Source` (or equivalent) so they end up in the same shape as hand-written code.

If those two things are true, the rule is satisfied. A subagent walking each line for "is this gofmt'd?" wastes time and produces noisy findings — a single `gofmt -l ./...` invocation answers the same question authoritatively.

## What the tooling subagent should verify

When this skill dispatches the tooling subagent for #06, it should check (in priority order):

1. **CI / presubmit** — is `gofmt -l` (or `goimports -l`, or `gofumpt -l`) run in CI such that unformatted code fails the build?
   - Look for: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`, `Makefile` targets, `pre-commit` config, `.golangci.yml` (with `gofmt` / `gofumpt` enabled)
2. **Pre-commit / editor** — is there a hook or editor config that runs the formatter on save?
   - Look for: `.pre-commit-config.yaml`, `.vscode/settings.json` with `editor.formatOnSave` + Go format tool, `.editorconfig`
3. **Generated code** — for any files marked `// Code generated ... DO NOT EDIT.`, is the generator using `format.Source` (or piping output through `gofmt`)?
   - Look for: generator scripts, `go:generate` directives, build pipelines invoking codegen tools

The subagent returns a single `pass / fail` (per check) and a one-line note pointing at the file that proved it (or its absence). No per-line code findings.

## Note on `gofumpt`

[`gofumpt`](https://github.com/mvdan/gofumpt) (Daniel Martí) is a **strict superset** of `gofmt`: it produces output that is also valid `gofmt` output, but adds extra rules `gofmt` chose not to enforce (e.g., consecutive `var` blocks merged into a single one, redundant `interface{}` simplified, single-statement compositions tightened, etc.).

The Style Guide does **not** mention `gofumpt`. It only mandates conformance to `gofmt`. So:

- A repo using only `gofmt` satisfies #06.
- A repo using `gofumpt` **also** satisfies #06 — because `gofumpt` output is, by construction, `gofmt`-compliant.
- A repo using **both** in conflicting ways (e.g., `gofmt` in CI but `gofumpt` in editor) will see flapping diffs; treat that as a **fail** even though each tool individually is fine. The subagent should flag the inconsistency.
- A repo using **neither** automatically (only manual formatting) — fail. The Guide is explicit about presubmit enforcement.

`gofumpt` is a reasonable upgrade for teams that want stricter, more uniform formatting beyond what `gofmt` enforces, but it is **not required** by Google's Style Guide. Treat it as a team choice.

## When NOT to fail

- The repo is small / personal / experimental, the user explicitly opts out of CI formatting checks. Note the deviation, don't escalate.
- Generated code that is `// DO NOT EDIT.` and is *intentionally* formatted by its generator (even if the output isn't strictly `gofmt`-clean) — defer to the generator's contract.
- Vendored third-party code under `/vendor/` or `/third_party/` — the formatting belongs to the upstream project.

## Suggested finding phrasing

- "No `gofmt` / `gofumpt` / `goimports` invocation found in CI — Style Guide #06 requires presubmit enforcement; add it to `.github/workflows/<file>.yml` (or equivalent)."
- "Pre-commit config exists at `.pre-commit-config.yaml` but does not include a Go formatter hook — add `gofmt` or `gofumpt`."
- "Generated file `proto/foo.pb.go` does not appear to be `gofmt`-formatted — pipe the generator output through `format.Source` or `gofmt`."
- "CI runs `gofmt` but editor settings invoke `gofumpt` — the two will produce different output for some constructs; pick one."

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Formatting section (the canonical text quoted above)
- <https://pkg.go.dev/cmd/gofmt> — `gofmt` reference
- <https://pkg.go.dev/go/format#Source> — `format.Source` (used to format generated code)
- <https://github.com/mvdan/gofumpt> — `gofumpt` (stricter superset; team choice, not required by Google)
