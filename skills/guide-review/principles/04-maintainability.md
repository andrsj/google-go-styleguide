---
principle: "Maintainability"
number: 4
category: philosophical
sources:
  - https://google.github.io/styleguide/go/guide
---

# #04 — Maintainability

## What it means

Maintainability is the **fourth** of Google's five Go style principles. It frames every other principle in terms of *time*: code is read and modified far more often than it's written, so today's choices need to keep tomorrow's edits safe and correct.

The Guide opens the section with the framing quote:

> Code is edited many more times than it is written. Readable code not only makes sense to a reader who is trying to understand how it works, but also to the programmer who needs to change it. Clarity is key.

Concrete characteristics of maintainable code per the Guide:

> Maintainable code:
>
> - Is easy for a future programmer to modify correctly
> - Has APIs that are structured so that they can grow gracefully
> - Is clear about the assumptions that it makes and chooses abstractions that map to the structure of the problem, not to the structure of the code
> - Avoids unnecessary coupling and doesn't include features that are not used
> - Has a comprehensive test suite to ensure promised behaviors are maintained and important logic is correct, and the tests provide clear, actionable diagnostics in case of failure

### Abstractions

> When using abstractions like interfaces and types which by definition remove information from the context in which they are used, it is important to ensure that they provide sufficient benefit.

### Hidden details

> Maintainable code also avoids hiding important details in places that are easy to overlook.

### Predictability

> Predictable names are another feature of maintainable code.

### Dependencies

> Maintainable code minimizes its dependencies (both implicit and explicit).

### Forward thinking

> When considering how to structure or write code, it is worth taking the time to think through ways in which the code may evolve over time.

## Why it's not a (direct) review rule

Maintainability is the **time-axis lens** reviewers apply, not a single-pattern detector. Concrete maintainability findings surface under more specific rules:

- **`decisions-review`** — interfaces, generics, type aliases, package-level state (each Style Decision pushes toward decoupling and minimum viable abstraction)
- **`best-practices-review`** — package size, function arguments, error handling, test patterns (the "comprehensive test suite with clear diagnostics" half of the definition)
- **#01 Clarity** — the "future reader" overlap
- **#02 Simplicity** — the "doesn't include features that are not used" overlap (YAGNI)
- **#03 Concision** — the "predictable names" overlap
- **#05 Consistency** — predictable, repeated patterns reduce the cost of every future edit

This file documents the *principle* so reviewers and other rules have a shared reference for what "maintainable" means in Google Go style.

## How reviewers should use it

Use Maintainability as background context when:

- A new abstraction (interface, generic, callback) is introduced — apply the Guide's test: does it provide *sufficient benefit*, or is it removing useful context for no current return?
- An API change makes today's call site nicer but pins the design in a way that's hard to extend — the Guide explicitly weights "grow gracefully" over "look nice now."
- A configuration value, side effect, or implicit dependency is being added in a place a future reader is unlikely to look (deep in `init`, hidden in a global, behind a build tag) — that's the Guide's "hiding important details" smell.
- A new dependency is being added — apply *minimize dependencies*. Even an internal-package dependency counts; the Guide names both implicit and explicit.
- A test exists but its failure message is opaque ("not equal", "expected true got false") — the Guide ties maintainability to *clear, actionable* test diagnostics, not just test presence.
- A name that *could* be predictable (matches existing patterns in the package) is instead novel — predictability is part of the definition.

A finding here usually belongs under a more specific rule. Use #04 in the *rationale* of a finding, not the rule label of one.

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Maintainability section (the canonical text quoted above)
