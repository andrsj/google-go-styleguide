---
principle: "Consistency"
number: 5
category: philosophical
sources:
  - https://google.github.io/styleguide/go/guide
---

# #05 — Consistency

## What it means

Consistency is the **fifth and last** of Google's five Go style principles. It's deliberately ranked at the bottom — not because it's unimportant, but because the four principles above it (Clarity, Simplicity, Concision, Maintainability) win when they conflict. Consistency is the **tie-breaker**.

Direct quotes from the Style Guide:

> Consistent code is code that looks, feels, and behaves like similar code throughout the broader codebase, within the context of a team or package, and even within a single file.

> Consistency concerns do not override any of the principles above, but if a tie must be broken, it is often beneficial to break it in favor of consistency.

> Consistency within a package is often the most immediately important level of consistency. It can be very jarring if the same problem is approached in multiple ways throughout a package, or if the same concept has many names within a file. However, even this should not override documented style principles or global consistency.

The Guide implies a layered hierarchy of consistency contexts:

1. **File** — names and approaches within a single file
2. **Package** — the most immediately important level (per the Guide)
3. **Team / module** — local conventions a team has settled on
4. **Codebase / global** — the wider Google Go codebase

Lower layers should match higher ones; when they conflict with documented style principles, the principles win.

## Why it's not a (direct) review rule

Consistency is the **comparison lens** reviewers apply, not a single-pattern detector. Whether a given snippet is "consistent" can only be judged against the surrounding code — the rule has no fixed shape. Concrete consistency findings emerge under more specific rules:

- **#10 Local consistency** — the explicit "follow local style unless it conflicts with the broader guide" rule (the actionable form of this principle within a file/package)
- **#07 MixedCaps**, **#09 Naming** — when a file/package mixes naming styles, the inconsistency is a real, point-able finding
- **`decisions-review`** — many decisions exist precisely to make consistency mechanical (initialisms, getter naming, error string casing, etc.)
- **`best-practices-review`** — package size, error handling, test patterns
- **#01 Clarity** — consistent code requires less context-switching to read
- **#02 Simplicity** — one approach everywhere means a simpler mental model than four parallel approaches
- **#03 Concision** — consistent idioms boost signal-to-noise; readers can skim familiar shapes
- **#04 Maintainability** — predictable patterns make every future edit safer

This file documents the *principle* so reviewers and other rules have a shared reference for how Google Go ranks consistency against the other four principles.

## How reviewers should use it

Use Consistency as background context when:

- Two valid styles both technically satisfy the higher principles — pick the one that matches the surrounding file/package.
- A change introduces a *new* way of doing something the package already does in another way (a second naming scheme for the same concept, a second error-handling pattern, a parallel helper that duplicates an existing one) — flag it.
- A team-local convention conflicts with the Style Guide — the Guide wins, but say so explicitly so the reviewee understands why their local pattern is being overridden.
- A reviewer is tempted to suggest a change "because that's how I'd write it" — only push if the change is supported by a higher principle. Pure preference doesn't justify breaking local consistency.
- A new file is the *first* file in a fresh package — there's nothing to be consistent with locally, so default to the broader Google style; future files in the package will then follow this one.

A finding here usually belongs under #10 Local consistency or a more specific rule. Use #05 in the *rationale* of a finding, not the rule label of one.

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Consistency section (the canonical text quoted above)
