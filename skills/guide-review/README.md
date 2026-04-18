# guide-review

Reviews Go code against [Google's Go Style Guide](https://google.github.io/styleguide/go/guide) — the canonical, normative ("non-negotiable") document of Google's three-part Go style framework.

This is one of three skills in the [`google-go-styleguide`](../../README.md) Claude Code plugin. It can be installed and used standalone, or as part of the full plugin alongside [`decisions-review`](../decisions-review/) and [`best-practices-review`](../best-practices-review/).

## What it covers

10 rules (5 principles + 5 guidelines):

**Principles** (philosophical — inform judgement, no findings emitted):

1. Clarity
2. Simplicity
3. Concision
4. Maintainability
5. Consistency

**Guidelines** (concrete conventions):

6. Formatting / `gofmt` (tooling — verifies the formatter is in CI / pre-commit / editor)
7. MixedCaps for multi-word identifiers (actionable)
8. Line length (actionable; includes a `## Tooling notes` subsection flagging `lll` as an anti-pattern)
9. Naming (actionable)
10. Local consistency (philosophical)

See [`SKILL.md`](SKILL.md) for the full review workflow and per-rule index.

## Install standalone

If you only want this one skill (without the rest of the `google-go-styleguide` plugin):

```bash
git clone https://github.com/andrsj/google-go-styleguide.git
cp -R google-go-styleguide/skills/guide-review ~/.claude/skills/
```

The skill then activates as `/guide-review` (un-namespaced).

## Install as part of the plugin

See the [top-level README](../../README.md) for `--plugin-dir` instructions. When loaded as a plugin, this skill is invoked as `/google-go-styleguide:guide-review`.

## When NOT to use this skill alone

`guide-review` covers only the top-level Style Guide principles and a handful of mandatory rules. The bulk of concrete review value lives in the sibling [`decisions-review`](../decisions-review/) skill (~50 specific decisions on naming, errors, language constructs, common libraries, and tests). For comprehensive Google-style review, install both.

## License

MIT — see the [top-level LICENSE](../../LICENSE).
