---
name: guide-review
description: Use when reviewing Go code against Google's Go Style Guide (the canonical, normative document). Activates only on explicit request ("review with Google Go Style Guide", "/google-go-styleguide:guide-review") — does NOT auto-trigger on `.go` files. Flags violations and groups findings by rule, each tied to a detailed file in `principles/` or `guidelines/`.
---

# Google Go Style Guide — Review

Code review skill based on Google's [Go Style Guide](https://google.github.io/styleguide/go/guide), the canonical and normative document of Google's three-part Go style framework. Each principle and guideline maps to a detailed rule file under `principles/` or `guidelines/`.

> [!NOTE]
> This skill covers the **Style Guide** only — the foundational, non-negotiable document. The companion skills `decisions-review` (concrete decisions) and `best-practices-review` (advisory patterns) cover the other two Google Go Style documents.

## When to use

- User explicitly asks for a "Google Go Style Guide review" or types `/google-go-styleguide:guide-review`
- User asks about a specific principle or guideline — load the matching file directly

This skill does **not** auto-activate on `.go` files. Google's Style Guide is opinionated, and the user should choose when to apply it.

## How to use

Reviews run as a **parallel fan-out of subagents**, one per actionable rule. This keeps the main context small and each check focused on a single principle.

> [!IMPORTANT]
> **Before any subagent fan-out**, ASK the user which model to use for the per-rule subagents: `opus` (highest quality, slowest), `sonnet` (balanced), or `haiku` (cheapest, fastest). Wait for their answer — do not pick for them. Pass the chosen model name to every dispatched subagent via the Task tool's `model` parameter.
>
> Skip this step only for the *Single-rule question* flow below — it loads one file directly in the main thread, no subagents.

### Full review

1. Identify the Go files in scope and hold only their paths in the main thread.
2. For each `actionable` rule in the index, dispatch a subagent with:
   - The rule number, short title, and path to its `.md` file
   - The list of `.go` file paths to review
   - Instruction: load only that one rule file, read the code, return findings in the format `<file>:<line> — <one-sentence observation>` or the literal string `no findings`
3. Dispatch all subagents **in parallel** (single message, multiple tool calls).
4. For `tooling` rules, dispatch a single subagent that verifies the tool is configured (e.g. `gofmt` is in CI / pre-commit / editor) and returns pass/fail — no code findings.
5. Skip `philosophical` rules entirely — they inform judgment, not findings.
6. Aggregate all subagent results, group by rule, and produce the final report (see *Output format*).

### Partial review

If the user asks to check against specific rules ("review for #07 and #09 only"), dispatch only those subagents. Same fan-out pattern, smaller set.

### Single-rule question

If the user asks about one rule specifically ("explain #01 Clarity" / "does this violate MixedCaps?"), skip the subagent fan-out: load the single rule file directly in the main thread and answer.

### Why this pattern

- Main context never holds more than one or two rule files at a time
- Each subagent reasons about exactly one rule — fewer false positives
- Parallel execution keeps wall-clock time low
- The user can always ask for a partial review to narrow scope further

## Categories

| Category | Meaning |
|---|---|
| `actionable` | Generates findings during review |
| `tooling` | Verify the tool is wired up, don't flag code |
| `philosophical` | Informs judgment, no findings emitted |

## Rule index

### Principles (philosophical foundation)

| # | Principle | Category | Rule file |
|---|---|---|---|
| 01 | Clarity | philosophical | [principles/01-clarity.md](principles/01-clarity.md) |
| 02 | Simplicity | philosophical | [principles/02-simplicity.md](principles/02-simplicity.md) |
| 03 | Concision | philosophical | [principles/03-concision.md](principles/03-concision.md) |
| 04 | Maintainability | philosophical | [principles/04-maintainability.md](principles/04-maintainability.md) |
| 05 | Consistency | philosophical | [principles/05-consistency.md](principles/05-consistency.md) |

### Guidelines (concrete conventions)

| # | Guideline | Category | Rule file |
|---|---|---|---|
| 06 | Formatting (gofmt) | tooling | [guidelines/06-formatting-gofmt.md](guidelines/06-formatting-gofmt.md) |
| 07 | MixedCaps for multi-word identifiers | actionable | [guidelines/07-mixedcaps.md](guidelines/07-mixedcaps.md) |
| 08 | Line length | actionable | [guidelines/08-line-length.md](guidelines/08-line-length.md) |
| 09 | Naming | actionable | [guidelines/09-naming.md](guidelines/09-naming.md) |
| 10 | Local consistency | philosophical | [guidelines/10-local-consistency.md](guidelines/10-local-consistency.md) |

## Output format

Group findings under each violated rule:

```
### Rule #N — <short title>
- path/to/file.go:42 — <one-sentence finding>
- path/to/file.go:87 — <one-sentence finding>
  See: <principles|guidelines>/NN-<slug>.md
```

End the review with a summary line: total findings, number of distinct rules violated, files touched.

## Source

- Style Guide: <https://google.github.io/styleguide/go/guide>
- Companion docs (covered by sibling skills): <https://google.github.io/styleguide/go/decisions>, <https://google.github.io/styleguide/go/best-practices>
