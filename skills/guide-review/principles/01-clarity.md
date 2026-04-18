---
principle: "Clarity"
number: 1
category: philosophical
sources:
  - https://google.github.io/styleguide/go/guide
---

# #01 — Clarity

## What it means

Clarity is the **first** of Google's five Go style principles, in priority order:

1. **Clarity** — the code's purpose and rationale is clear to the reader
2. Simplicity
3. Concision
4. Maintainability
5. Consistency

Direct quotes from the Style Guide:

> The core goal of readability is to produce code that is clear to the reader.

> Clarity is primarily achieved with effective naming, helpful commentary, and efficient code organization.

> Clarity is to be viewed through the lens of the reader, not the author of the code.

> It is more important that code be easy to read than easy to write.

The Guide splits clarity into two facets:

> Clarity in code has two distinct facets: What is the code actually doing? Why is the code doing what it does?

### What is the code actually doing?

> Go is designed such that it should be relatively straightforward to see what the code is doing.

When code is hard to follow, the Guide prescribes:

- More descriptive variable names
- Additional commentary
- Whitespace and comments to break up dense blocks
- Refactoring into separate functions/methods for modularity

> There is no one-size-fits-all approach here, but it is important to prioritize clarity when developing Go code.

### Why is the code doing what it does?

> The code's rationale is often sufficiently communicated by the names of variables, functions, methods, or packages. Where it is not, it is important to add commentary.

> It is often better for comments to explain why something is done, not what the code is doing.

This is *especially* important "when the code contains nuances that a reader may not be familiar with."

## Why it's not a (direct) review rule

Clarity is the **lens** reviewers apply, not a single-pattern detector. A subagent that scans every Go file for "clarity violations" would fire on almost any code and produce false positives. Concrete clarity-related findings surface under more specific rules in:

- **#07 MixedCaps** — naming convention violations
- **#09 Naming** — non-descriptive or repetitive names
- **`decisions-review`** — early returns, doc comments, exported-symbol comments, etc.
- **`best-practices-review`** — concrete clarity patterns (avoid `if-else` pyramids, name intermediate steps, etc.)
- **#02 Simplicity** — overlaps in pushing back on dense one-liners; clarity is reader-focused, simplicity is implementation-focused
- **#03 Concision** — overlaps in surfacing salient details; concision targets signal-to-noise specifically
- **#04 Maintainability** — clarity is the time-projected payoff (the future reader)
- **#05 Consistency** — repeated patterns aid comprehension; consistent code is clearer

This file documents the *principle* so reviewers (and other rules) have a shared reference for what "clarity" means in Google Go style.

## How reviewers should use it

Use Clarity as background context when:

- Judging borderline cases under more specific rules — when in doubt, ask "would this be clearer to a new reader?"
- Reviewing comment quality — does the comment explain *why*, or just restate *what* the code already shows?
- Evaluating naming choices — does the name describe purpose, or just type / position?
- Considering structural refactors — would extracting a function or adding whitespace make the next reader's job easier?

A finding here usually belongs under a more specific rule. Use #01 in the *rationale* of a finding, not the rule label of one.

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Clarity section (the canonical text quoted above)
