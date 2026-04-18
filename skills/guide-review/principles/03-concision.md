---
principle: "Concision"
number: 3
category: philosophical
sources:
  - https://google.github.io/styleguide/go/guide
  - https://google.github.io/styleguide/go/best-practices
  - https://go.dev/wiki/TableDrivenTests
---

# #03 — Concision

## What it means

Concision is the **third** of Google's five Go style principles (after Clarity and Simplicity). It targets *signal-to-noise ratio* — how easily a reader can pick out the parts of the code that actually matter from everything else on the page.

Direct quotes from the Style Guide:

> Concise Go code has a high signal-to-noise ratio.

> It is easy to discern the relevant details, and the naming and structure guide the reader through these details.

The Guide names five things that **lower** the signal-to-noise ratio:

> There are many things that can get in the way of surfacing the most salient details at any given time:
>
> - Repetitive code
> - Extraneous syntax
> - Opaque names
> - Unnecessary abstraction
> - Whitespace

### Repetitive code

The Guide singles this out as especially harmful:

> Repetitive code especially obscures the differences between each nearly-identical section, and requires a reader to visually compare similar lines of code to find the changes.

> Table-driven testing is a good example of a mechanism that can concisely factor out the common code from the important details of each repetition, but the choice of which pieces to include in the table will have an impact on how easy the table is to understand.

The implicit warning: factoring shared code into a table or helper is good *only if* it surfaces the per-case differences clearly. A poorly-chosen abstraction can hide the very details it was meant to highlight.

### Structure follows signal

> When considering multiple ways to structure code, it is worth considering which way makes important details the most apparent.

> Understanding and using common code constructions and idioms are also important for maintaining a high signal-to-noise ratio.

### Signal boosting

When code looks like a familiar idiom but is subtly different, the Guide endorses *intentionally* breaking the pattern enough that a reader notices:

> If code looks very similar to this but is subtly different, a reader may not notice the change. In cases like this, it is worth intentionally "boosting" the signal of the error check by adding a comment to call attention to it.

(See the [signal-boost section](https://google.github.io/styleguide/go/best-practices#signal-boost) in Best Practices for the canonical example.)

## Why it's not a (direct) review rule

Concision is a *design lens*, not a single-pattern detector. Concrete concision-related findings show up under more specific rules:

- **`decisions-review`** — repetition (don't repeat package name in identifiers), commentary, naming
- **`best-practices-review`** — table-driven tests, signal boost, package size, error-handling patterns
- **#01 Clarity** — overlaps but is reader-focused; concision is structure-focused
- **#02 Simplicity** — overlaps but is implementation-focused; concision is expression-focused
- **#04 Maintainability** — high signal-to-noise code is faster to revisit and modify
- **#05 Consistency** — using the same idiom each time means less context to load on every read; consistent code is more concise *in the reader's head*

This file documents the *principle* so reviewers and other rules have a shared reference for what "concise" means in Google Go style.

## How reviewers should use it

Use Concision as background context when:

- A change introduces near-duplicate blocks (test cases, switch arms, error returns) — would a table or helper surface the per-case differences better, or would it hide them?
- A change adds an abstraction that "feels" right but doesn't actually clarify the per-call differences — the Guide warns this can lower signal.
- Whitespace or comment density seems off — too tight makes it dense; too loose dilutes the signal. The Guide includes whitespace in its list of noise sources.
- A line of code looks like a common idiom but does something subtly different — recommend a *signal boost* (a clarifying comment, a renamed helper, an intentional break in the pattern).
- A name is "opaque" (single-letter outside a tiny scope, ambiguous abbreviation, generic word like `data` / `info`) — fold the finding into #09 Naming, but the Concision lens is what flagged it.

A finding here usually belongs under a more specific rule. Use #03 in the *rationale* of a finding, not the rule label of one.

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Concision section (the canonical text quoted above)
- <https://google.github.io/styleguide/go/best-practices#signal-boost> — Signal boost pattern
- <https://go.dev/wiki/TableDrivenTests> — Table-driven testing (referenced as the canonical concision-via-factoring example)
