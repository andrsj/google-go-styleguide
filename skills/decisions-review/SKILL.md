---
name: decisions-review
description: Use when reviewing Go code against Google's Go Style Decisions (the normative-but-subordinate document of opinionated style choices — naming, commentary, imports, errors, language constructs, common libraries, and test failures). Activates only on explicit request ("review with Google Go Style Decisions", "/google-go-styleguide:decisions-review") — does NOT auto-trigger on `.go` files. Findings are grouped by section, each tied to a detailed file in `sections/`.
---

# Google Go Style Decisions — Review

Code review skill based on Google's [Go Style Decisions](https://google.github.io/styleguide/go/decisions) — the second of Google's three Go style documents. Decisions is **normative but not canonical**: it's where Google readability mentors apply most concrete feedback, and it's subordinate to the [Style Guide](https://google.github.io/styleguide/go/guide) when conflicts arise.

> [!NOTE]
> This skill covers the **Style Decisions** only. The companion skills `guide-review` (the canonical foundational document) and `best-practices-review` (advisory patterns) cover the other two Google Go Style documents.

## When to use

- User explicitly asks for a "Google Go Style Decisions review" or types `/google-go-styleguide:decisions-review`
- User asks about a specific decision section ("how should I name receivers?" / "review my error handling against Google's decisions")
- User asks about a specific rule by anchor (e.g., `#initialisms`, `#in-band-errors`) — load the matching section file and answer

This skill does **not** auto-activate on `.go` files. Decisions is opinionated, and the user should choose when to apply it.

## How to use

Reviews run as a **parallel fan-out of subagents**, one per *section* (not one per individual rule). Each section file groups all the related decisions for one topic — so each subagent reasons coherently about (e.g.) all naming decisions together, instead of fragmenting tightly coupled rules across separate dispatches.

### Full review

1. Identify the Go files in scope and hold only their paths in the main thread.
2. For each `actionable` section in the index, dispatch a subagent with:
   - The section number, name, and path to its `sections/NN-*.md` file
   - The list of `.go` file paths to review
   - Instruction: load only that one section file, read the code, return findings in the format `<file>:<line> — <rule-anchor> — <one-sentence observation>` or the literal string `no findings`
3. Dispatch all subagents **in parallel** (single message, multiple tool calls).
4. Aggregate all subagent results, group by section (and by rule-anchor within each section), and produce the final report (see *Output format*).

### Partial review

If the user asks to check against a specific section ("review for naming and errors only") or specific rules ("only check `#getters` and `#in-band-errors`"), dispatch only the relevant section subagents — and instruct each subagent to focus on the requested rules within its file.

### Single-rule question

If the user asks about one rule specifically ("explain `#receiver-names`" / "does this code violate `#dot-imports`?"), skip the subagent fan-out: load the single section file directly in the main thread and answer.

### Why this pattern (sections, not individual rules)

- Decisions has ~65 individual rules; one subagent per rule is too much fan-out.
- Rules within a section are tightly coupled (all naming rules feed each other; all error-handling decisions cohere). A section subagent reasons about the whole topic together — fewer false positives, better cross-rule findings.
- Each section file still indexes every individual rule with anchor links to the source doc, so partial / single-rule queries work.
- Main context never holds more than one or two section files at a time.
- Parallel execution keeps wall-clock time low.

## Categories

| Category | Meaning |
|---|---|
| `actionable` | Generates findings during review |
| `philosophical` | Informs judgment, no findings emitted |

(Decisions has no `tooling` rules — the only tooling rule in the Style Guide framework is `#06 Formatting / gofmt`, covered by sibling skill `guide-review`.)

## Section index

Every section file covers all the decisions in that section, with anchors matching the source doc. Use `<file>#<rule-anchor>` to reference a specific decision.

### 01 — Naming (12 rules) — actionable

[`sections/01-naming.md`](sections/01-naming.md)

- `#underscores` — Underscores
- `#package-names` — Package names
- `#receiver-names` — Receiver names
- `#constant-names` — Constant names
- `#initialisms` — Initialisms
- `#getters` — Getters
- `#variable-names` — Variable names
- Single-letter variable names
- `#repetition` — Repetition
- Package vs. exported symbol name
- Variable name vs. type
- External context vs. local names

### 02 — Commentary (6 rules) — actionable

[`sections/02-commentary.md`](sections/02-commentary.md)

- `#comment-line-length` — Comment line length
- `#doc-comments` — Doc comments
- `#comment-sentences` — Comment sentences
- `#examples` — Examples
- `#named-result-parameters` — Named result parameters
- `#package-comments` — Package comments

### 03 — Imports (4 rules) — actionable

[`sections/03-imports.md`](sections/03-imports.md)

- `#import-renaming` — Import renaming
- `#import-grouping` — Import grouping
- `#import-blank` — Import "blank"
- `#import-dot` — Import "dot"

### 04 — Errors (5 rules) — actionable

[`sections/04-errors.md`](sections/04-errors.md)

- `#returning-errors` — Returning errors
- `#error-strings` — Error strings
- `#handle-errors` — Handle errors
- `#in-band-errors` — In-band errors
- `#indent-error-flow` — Indent error flow

### 05 — Language (23 rules) — actionable

[`sections/05-language.md`](sections/05-language.md)

- `#literal-formatting` — Literal formatting
- Field names
- `#literal-matching-braces` — Matching braces
- `#cuddled-braces` — Cuddled braces
- Repeated type names
- `#zero-value-fields` — Zero-value fields
- `#nil-slices` — Nil slices
- `#indentation-confusion` — Indentation confusion
- `#func-formatting` — Function formatting
- `#conditional-formatting` — Conditionals and loops
- `#copying` — Copying
- `#dont-panic` — Don't panic
- `#must-functions` — Must functions
- `#goroutine-lifetimes` — Goroutine lifetimes
- `#interfaces` — Interfaces
- `#generics` — Generics
- `#pass-values` — Pass values
- `#receiver-type` — Receiver type
- `#switch-and-break` — Switch and break
- `#synchronous-functions` — Synchronous functions
- `#type-aliases` — Type aliases
- `#use-q` — Use %q
- `#use-any` — Use any

### 06 — Common libraries (5 rules) — actionable

[`sections/06-common-libraries.md`](sections/06-common-libraries.md)

- `#flags` — Flags
- `#logging` — Logging
- `#contexts` — Contexts
- Custom contexts
- `#cryptorand` — crypto/rand

### 07 — Useful test failures (9 rules) — actionable

[`sections/07-test-failures.md`](sections/07-test-failures.md)

- `#assertion-libraries` — Assertion libraries
- `#identify-the-function` — Identify the function
- `#identify-the-input` — Identify the input
- `#got-before-want` — Got before want
- `#full-structure-comparisons` — Full structure comparisons
- `#compare-stable-results` — Compare stable results
- `#keep-going` — Keep going
- `#types-of-equality` — Equality comparison and diffs
- `#level-of-detail` — Level of detail

## Output format

Group findings under each section, with the rule anchor noted on each finding:

```
### Section #N — <section name>
- path/to/file.go:42 — `#rule-anchor` — <one-sentence finding>
- path/to/file.go:87 — `#other-anchor` — <one-sentence finding>
  See: sections/NN-<slug>.md#<rule-anchor>
```

End the review with a summary line: total findings, number of distinct rules violated, files touched.

## Source

- Style Decisions: <https://google.github.io/styleguide/go/decisions>
- Companion docs (covered by sibling skills): <https://google.github.io/styleguide/go/guide>, <https://google.github.io/styleguide/go/best-practices>
