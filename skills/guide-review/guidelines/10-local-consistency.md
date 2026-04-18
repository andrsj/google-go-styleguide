---
guideline: "Local consistency"
number: 10
category: philosophical
sources:
  - https://google.github.io/styleguide/go/guide
---

# #10 — Local consistency

## What it requires

This guideline tells reviewers (and authors) **when local file/package style takes precedence** over personal preference, and **when a local style deviation is no longer a valid excuse** for new code that would entrench or expand it.

The Style Guide's framing:

> Where the style guide has nothing to say about a particular point of style, authors are free to choose the style that they prefer, unless the code in close proximity (usually within the same file or package, but sometimes within a team or project directory) has taken a consistent stance on the issue.

So the default precedence chain when the Guide is silent:

1. Local file → match it
2. Package → match it
3. Team / project directory → match it
4. Otherwise — author's choice

### Valid vs invalid scope of "local style"

The Guide gives concrete examples on **both** sides.

**Valid** local style considerations (the Guide is silent → local consistency wins):

> - Use of `%s` or `%v` for formatted printing of errors
> - Usage of buffered channels in lieu of mutexes

**Invalid** local style considerations (the Guide *does* speak → no amount of local consistency justifies violating it):

> - Line length restrictions for code
> - Use of assertion-based testing libraries

The pattern: if a topic is governed by the Style Guide, Style Decisions, or Best Practices, the document wins. Local convention only fills the gaps.

### When existing deviations live, and when they must be fixed

The Guide acknowledges that existing files may already deviate from the documented style. It then draws a clear line for *new* code:

> If the local style disagrees with the style guide but the readability impact is limited to one file, it will generally be surfaced in a code review for which a consistent fix would be outside the scope of the CL in question.

> At that point, it is appropriate to file a bug to track the fix.

> If a change would worsen an existing style deviation, expose it in more API surfaces, expand the number of files in which the deviation is present, or introduce an actual bug, then local consistency is no longer a valid justification for violating the style guide for new code.

> In these cases, it is appropriate for the author to clean up the existing codebase in the same CL, perform a refactor in advance of the current CL, or find an alternative that at least does not make the local problem worse.

In other words, the boundary is **expansion**:

| Situation | Action |
|---|---|
| Existing in-file deviation, change touches it but doesn't expand it | File a bug, leave the fix out of scope |
| Change *worsens* the deviation (more files, more API surface, more callers) | No longer a free pass — fix in this CL, refactor first, or find a non-expanding alternative |
| Change introduces a new deviation in greenfield code | Not allowed; the Guide wins |

## Why it's a philosophical (not actionable) rule

Local consistency is **meta**: it doesn't say what code should look like, it says *whose preference wins when the documented rules are silent*. A subagent can't lint for "is this consistent with the rest of the file?" without knowing what convention the file/package has settled on — that's project-specific judgment.

But the *meta-rule* itself ("don't expand existing deviations in new code") is a clear principle a reviewer can apply consistently. Concrete violations show up under more specific rules:

- **#05 Consistency** — the principle this guideline is the actionable form of
- **#07 MixedCaps**, **#09 Naming** — when a file mixes naming styles, that's a real, point-able finding
- **`decisions-review`** — many decisions exist precisely because the Guide chose not to leave them to local convention
- **`best-practices-review`** — pragmatic patterns that often defer to local style

## How reviewers should use it

Use #10 as the framework when you spot a file/package style choice and need to decide whether to enforce it on new code:

- **Is the Guide / Decisions / Best Practices silent on this point?**
  - Yes → local style wins. Match the file. Don't push the reviewee to do it your preferred way.
  - No → the document wins. Note the deviation politely; cite the source.
- **Is the change adding a new instance of an existing deviation?**
  - No (the change is neutral or fixes it) → file a bug for the cleanup if not done in this CL.
  - Yes (the change spreads the deviation: new file, new API caller, new package importing it, more lines following the bad pattern) → block the expansion. The reviewee should fix in this CL, refactor first, or find a non-expanding alternative.
- **Is the deviation legitimately scoped to a single file with no public surface impact?** → tolerate, file a bug.
- **Has the deviation leaked into a public API or a widely-imported helper?** → that's exactly the case where the Guide says "no longer a valid justification" — flag the leak.

A finding here usually surfaces as part of a more specific rule's finding (the *what* lives there; #10 is the *why this CL can't expand it*). When using #10 directly, name the local convention and the documented rule it's competing with.

## Suggested phrasing

When a deviation exists but the change doesn't worsen it:

> "This file uses `%v` for error formatting throughout; matches local convention. No change needed in this CL — the broader Style Guide is silent on `%s` vs `%v` for errors. (See #10.)"

When a change *would* worsen an existing deviation:

> "Existing file uses `assert.Equal(...)` (assertion-library style), which the Style Guide flags as **not** a valid local style. This CL adds 12 new `assert.*` calls — that expands the deviation. Per #10, please refactor existing tests to standard `if got != want { t.Errorf(...) }` in this CL, or split the change so this one doesn't grow the surface."

When greenfield code introduces a new deviation:

> "New file, no local convention to defer to. The pattern here violates Style Guide §<X>; please follow the documented rule. (See #10 — local consistency only applies when the Guide is silent.)"

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, Local consistency section (the canonical text quoted above)
