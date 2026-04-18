---
principle: "Simplicity"
number: 2
category: philosophical
sources:
  - https://google.github.io/styleguide/go/guide
---

# #02 — Simplicity

## What it means

Simplicity is the **second** of Google's five Go style principles (after Clarity, before Concision). Where Clarity is about the *reader*, Simplicity is about the *implementation*: write the smallest, most direct thing that accomplishes the goal — and only add complexity deliberately when it's actually required.

Direct quotes from the Style Guide:

> Your Go code should be simple for those using, reading, and maintaining it.

> Go code should be written in the simplest way that accomplishes its goals, both in terms of behavior and performance.

The Guide gives a concrete checklist of what "simple code" looks like in the Google Go codebase:

> Within the Google Go codebase, simple code:
> - Is easy to read from top to bottom
> - Does not assume that you already know what it is doing
> - Does not assume that you can memorize all of the preceding code
> - Does not have unnecessary levels of abstraction
> - Does not have names that call attention to something mundane
> - Makes the propagation of values and decisions clear to the reader
> - Has comments that explain why, not what, the code is doing to avoid future deviation
> - Has documentation that stands on its own
> - Has useful errors and useful test failures
> - May often be mutually exclusive with 'clever' code

On adding complexity:

> Tradeoffs can arise between code simplicity and API usage simplicity.

> When code needs complexity, the complexity should be added deliberately.

> This principle does not imply that complex code cannot or should not be written in Go.

> If code turns out to be very complex when its purpose should be simple, this is often a signal to revisit the implementation.

### Least mechanism

A subprinciple that turns Simplicity into a concrete tie-breaker:

> Where there are several ways to express the same idea, prefer the one that uses the most standard tools.

> It is easy to add complexity to code as needed, whereas it is much harder to remove existing complexity.

In practice this means: prefer stdlib over a custom helper, prefer a slice over a custom collection, prefer a `for` loop over a generic abstraction — until the standard tool actually fails to fit.

## Why it's not a (direct) review rule

Simplicity is the **disposition** reviewers bring, not a single-pattern detector. Like Clarity, a "simplicity violation" subagent would fire on too much code with too many false positives. The concrete violations show up under more specific rules:

- **`decisions-review`** — generics, interfaces, type aliases, panics (each has its own "use sparingly / only when X" decision)
- **`best-practices-review`** — package size, function arguments, variable declarations, complex CLIs (concrete patterns for keeping things small)
- **#01 Clarity** — the Guide pairs them tightly ("Clarity is key" appears even inside the Maintainability section); simple code is usually clearer code
- **#03 Concision** — closely related, but Simplicity is about *what's there*; Concision is about *how it's expressed*
- **#04 Maintainability** — long-term complexity cost
- **#05 Consistency** — when two patterns are equally simple, pick the one that matches the surrounding code

This file documents the *principle* so reviewers and other rules have a shared reference for what "simple" means in Google Go style.

## How reviewers should use it

Use Simplicity as background context when:

- A change adds a new abstraction layer (interface, generic type, callback, builder) — ask: does this complexity earn its keep, or could a direct implementation work today?
- A change introduces a third-party dependency for something the stdlib already supports — apply *least mechanism*.
- A function's purpose sounds simple ("parse a config", "send a notification") but the implementation has grown branchy — that's the Guide's "signal to revisit" trigger.
- An API change makes the implementation simpler at the cost of a more confusing public surface (or vice-versa) — the Guide explicitly flags this tradeoff.
- A reviewer is tempted to suggest a "clever" refactor — the Guide is explicit that simple "may often be mutually exclusive with 'clever' code"; bias toward the simple version.

A finding here usually belongs under a more specific rule. Use #02 in the *rationale* of a finding, not the rule label of one.

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Simplicity section + "Least mechanism" subsection (the canonical text quoted above)
