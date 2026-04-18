# decisions-review

Reviews Go code against [Google's Go Style Decisions](https://google.github.io/styleguide/go/decisions) — the **normative-but-subordinate** document of opinionated, concrete style choices. This is where Google readability mentors apply most feedback.

This is one of three skills in the [`google-go-styleguide`](../../README.md) Claude Code plugin. It can be installed and used standalone, or as part of the full plugin alongside [`guide-review`](../guide-review/) and [`best-practices-review`](../best-practices-review/).

## What it covers

7 sections covering ~65 individual rules. The skill organizes rules **per section** (not per individual rule), so each section subagent reasons coherently about a related topic:

| # | Section | Rules |
|---|---|---|
| 01 | Naming | 9 + 3 sub-rules under `#repetition` (underscores, package names, receivers, constants, initialisms, getters, variables, single-letter, repetition) |
| 02 | Commentary | 6 (line length, doc comments, sentences, examples, named results, package comments) |
| 03 | Imports | 4 (renaming, grouping, blank, dot) |
| 04 | Errors | 5 (returning, strings, handling, in-band, indent flow) |
| 05 | Language | 23 (literal formatting, braces ×2, fields, repeated types, zero-value, nil slices, indentation, func formatting, conditionals, copying, panic, Must, goroutines, interfaces, generics, pass values, receivers, switch/break, sync, type aliases, %q, any) |
| 06 | Common libraries | 5 (flags, logging, contexts, custom contexts, crypto/rand) |
| 07 | Useful test failures | 9 (assertion libraries, identify fn, identify input, got/want, full struct, stable results, keep going, equality, detail level) |

See [`SKILL.md`](SKILL.md) for the full review workflow and per-section index.

## Why per-section, not per-rule

Decisions has ~65 individual rules; one subagent per rule would be too much fan-out. Rules within a section are tightly coupled (all naming rules feed each other; all error-handling decisions cohere). A section subagent reasons about the whole topic together — fewer false positives, better cross-rule findings. Each section file still indexes every individual rule with anchor links to the source doc, so partial / single-rule queries work.

## Install standalone

If you only want this one skill:

```bash
git clone https://github.com/andrsj/google-go-styleguide.git
cp -R google-go-styleguide/skills/decisions-review ~/.claude/skills/
```

Activates as `/decisions-review` (un-namespaced).

## Install as part of the plugin

See the [top-level README](../../README.md) for `--plugin-dir` instructions. When loaded as a plugin, this skill is invoked as `/google-go-styleguide:decisions-review`.

## Precedence

Decisions is **subordinate to the Style Guide**. Where they conflict, Style Guide wins. The sibling [`guide-review`](../guide-review/) skill covers the canonical document; install it alongside this one for the full hierarchy.

## License

MIT — see the [top-level LICENSE](../../LICENSE).
