---
name: best-practices-review
description: Use when reviewing Go code against Google's Go Best Practices (the auxiliary, non-canonical advisory document with pragmatic guidance for naming, error handling, documentation, variable declarations, function argument lists, complex CLIs, and tests). Activates only on explicit request ("review with Google Go Best Practices", "/google-go-styleguide:best-practices-review") — does NOT auto-trigger on `.go` files. Findings are advisory ("should") rather than prescriptive ("must"); grouped by section, each tied to a detailed file in `sections/`.
---

# Google Go Best Practices — Review

Code review skill based on Google's [Go Best Practices](https://google.github.io/styleguide/go/best-practices) — the third of Google's three Go style documents. Best Practices is **neither normative nor canonical**: it's auxiliary guidance that "may not apply in every circumstance." Findings here are recommendations, not rule violations.

> [!NOTE]
> This skill covers **Best Practices** only. The companion skills `guide-review` (canonical Style Guide) and `decisions-review` (concrete Style Decisions) cover the other two Google Go Style documents. Where Best Practices conflicts with them, the higher-precedence document wins:
>
> **Style Guide > Style Decisions > Best Practices**

## When to use

- User explicitly asks for a "Google Go Best Practices review" or types `/google-go-styleguide:best-practices-review`
- User asks about a specific best-practice section ("review my error handling against Best Practices", "how should I structure my variable declarations?")
- User asks about a specific anchor (e.g., `#error-structure`, `#option-structure`) — load the matching section file directly

This skill does **not** auto-activate on `.go` files. Best Practices is the most opinionated and most context-dependent of the three docs; the user should choose when to apply it.

## How to use

Same pattern as the sibling `decisions-review` skill: **parallel fan-out of subagents, one per section**. Each section file groups all related practices under sub-headings.

### Full review

1. Identify the Go files in scope and hold only their paths in the main thread.
2. For each `actionable` section in the index, dispatch a subagent with:
   - The section number, name, and path to its `sections/NN-*.md` file
   - The list of `.go` file paths to review
   - Instruction: load only that one section file, read the code, return findings in the format `<file>:<line> — `<rule-anchor>` — <one-sentence advisory>` or the literal string `no findings`. Phrase findings as **suggestions**, not violations.
3. Dispatch all subagents **in parallel** (single message, multiple tool calls).
4. Aggregate all subagent results, group by section, and produce the final report.

### Partial review

If the user asks to check a specific section ("only error handling and tests"), dispatch only the relevant section subagents.

### Single-rule question

For "explain `#option-structure`" / "does this violate `#error-structure`?", load the single section file directly and answer.

### Why this is advisory, not prescriptive

The Best Practices doc explicitly opens with: *"This guidance is intended for common situations encountered while authoring Go in the Google codebase. It may not apply in every circumstance."* So reports from this skill should phrase findings as recommendations, not violations. A finding here can be discussed and overruled by project context — that's expected.

If the same code is also violating `guide-review` or `decisions-review`, those skills' findings take precedence.

## Categories

| Category | Meaning |
|---|---|
| `actionable` | Generates findings during review (advisory) |
| `philosophical` | Informs judgment, no findings emitted |

## Section index

### 01 — Naming (4 sub-rules) — actionable

[`sections/01-naming.md`](sections/01-naming.md)

- Function and method names (with sub-sub-rules: avoid repetition, naming conventions)
- Test double and helper packages (with sub-sub-rules: creating helpers, simple case, multiple behaviors, multiple doubles, local variables in tests)
- Shadowing
- Util packages

### 02 — Package size (single section) — philosophical

[`sections/02-package-size.md`](sections/02-package-size.md)

### 03 — Imports (2 sub-rules) — actionable

[`sections/03-imports.md`](sections/03-imports.md)

- Protocol Buffer Messages and Stubs
- Import ordering

### 04 — Error handling (7 sub-rules) — actionable

[`sections/04-error-handling.md`](sections/04-error-handling.md)

- Error structure
- Adding information to errors
- Placement of `%w` in errors (with sub-rule: Sentinel error placement)
- Logging errors (with sub-rule: Custom verbosity levels)
- Program initialization
- Program checks and panics
- When to panic

### 05 — Documentation (4 sub-rules) — actionable

[`sections/05-documentation.md`](sections/05-documentation.md)

- Conventions (with sub-sub-rules: parameters/config, contexts, concurrency, cleanup, errors)
- Preview
- Godoc formatting
- Signal boosting

### 06 — Variable declarations (5 sub-rules) — actionable

[`sections/06-variable-declarations.md`](sections/06-variable-declarations.md)

- Initialization
- Declaring variables with zero values
- Composite literals
- Size hints
- Channel direction

### 07 — Function argument lists (2 sub-rules) — actionable

[`sections/07-function-arguments.md`](sections/07-function-arguments.md)

- Option structure
- Variadic options

### 08 — Complex command-line interfaces (single section) — actionable

[`sections/08-complex-clis.md`](sections/08-complex-clis.md)

### 09 — Tests (8 sub-rules) — actionable

[`sections/09-tests.md`](sections/09-tests.md)

- Leave testing to the `Test` function
- Designing extensible validation APIs (with sub-rules: acceptance testing, writing an acceptance test)
- Use real transports
- `t.Error` vs. `t.Fatal`
- Error handling in test helpers
- Don't call `t.Fatal` from separate goroutines
- Use field names in struct literals
- Keep setup code scoped to specific tests (with sub-rules: when to use a custom `TestMain`, amortizing common test setup)

### 10 — String concatenation (4 sub-rules) — actionable

[`sections/10-string-concatenation.md`](sections/10-string-concatenation.md)

- Prefer `+` for simple cases
- Prefer `fmt.Sprintf` when formatting
- Prefer `strings.Builder` for constructing a string piecemeal
- Constant strings

### 11 — Global state (3 sub-rules + intro) — actionable

[`sections/11-global-state.md`](sections/11-global-state.md)

- Major forms of package state APIs
- Litmus tests
- Providing a default instance

### 12 — Interfaces (3 sub-rules + intro) — actionable

[`sections/12-interfaces.md`](sections/12-interfaces.md)

- Avoid unnecessary interfaces
- Interface ownership and visibility
- Designing effective interfaces

## Output format

Group findings under each section, with the rule anchor noted on each finding. Phrase findings as **suggestions**, not violations:

```
### Section #N — <section name>
- path/to/file.go:42 — `#rule-anchor` — <suggestion phrased as recommendation>
- path/to/file.go:87 — `#other-anchor` — <suggestion>
  See: sections/NN-<slug>.md#<rule-anchor>
```

End the review with a summary line: total suggestions, number of distinct rules referenced, files touched. Note explicitly that these are advisory (Best Practices is non-canonical); the team may have valid reasons to deviate.

## Source

- Best Practices: <https://google.github.io/styleguide/go/best-practices>
- Companion docs (covered by sibling skills): <https://google.github.io/styleguide/go/guide>, <https://google.github.io/styleguide/go/decisions>
