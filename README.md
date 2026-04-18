# google-go-styleguide

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) bundling three skills that review Go code against [Google's Go Style Guide framework](https://google.github.io/styleguide/go/).

Each skill maps to one of Google's three official Go style documents and runs as a **parallel fan-out of subagents** — one per rule (or per section, for the bigger doc) — so reviews stay focused and the main context stays small. All three skills are **explicit-only** (no auto-trigger on `.go` files); you choose when to apply Google's opinionated rules.

## The three skills

| # | Skill | Source | Invocation | Coverage |
|---|---|---|---|---|
| 1 | `guide-review` | [Style Guide](https://google.github.io/styleguide/go/guide) — canonical, normative ("non-negotiable") | `/google-go-styleguide:guide-review` | 5 principles + 5 guidelines (gofmt, MixedCaps, line length, naming, local consistency) |
| 2 | `decisions-review` | [Style Decisions](https://google.github.io/styleguide/go/decisions) — normative-but-subordinate concrete decisions | `/google-go-styleguide:decisions-review` | 7 sections × ~50 rules (naming, commentary, imports, errors, language, common libraries, useful test failures) |
| 3 | `best-practices-review` | [Best Practices](https://google.github.io/styleguide/go/best-practices) — auxiliary advisory patterns | `/google-go-styleguide:best-practices-review` | 12 sections × ~50 sub-rules (naming, package size, imports, error handling, documentation, variable declarations, function arguments, complex CLIs, tests, string concatenation, global state, interfaces) |

The split mirrors how Google itself frames the docs:

> Style Guide > Style Decisions > Best Practices

Where they conflict, the higher-precedence document wins. The skills inherit that hierarchy — `guide-review` flags hard rules, `decisions-review` flags concrete choices, `best-practices-review` flags soft recommendations.

## Install

This repo doubles as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) (`.claude-plugin/marketplace.json`) — so it's also installable via the official `/plugin install` flow. Three install paths, in order of recommendation:

### Option 1 — Via the marketplace (recommended)

Inside any Claude Code session, run:

```text
/plugin marketplace add andrsj/google-go-styleguide
/plugin install google-go-styleguide@andrsj-skills
/reload-plugins
```

The plugin is registered globally (user scope by default), available in every project. Skills appear namespaced as `/google-go-styleguide:guide-review`, `/google-go-styleguide:decisions-review`, `/google-go-styleguide:best-practices-review`. Updates flow automatically when you run `/plugin marketplace update andrsj-skills`.

### Option 2 — `--plugin-dir` (for development / live-edit)

If you've cloned the repo locally and want to iterate on skill files with instant feedback, start Claude Code with `--plugin-dir` pointing at the clone:

```bash
git clone https://github.com/andrsj/google-go-styleguide.git
claude --plugin-dir ./google-go-styleguide
```

Edits to any skill file pick up live with `/reload-plugins` — no re-install needed.

### Option 3 — Standalone copy (one skill, no namespace)

If you only want one of the skills and don't care about the plugin namespace:

```bash
git clone https://github.com/andrsj/google-go-styleguide.git
cp -R google-go-styleguide/skills/guide-review ~/.claude/skills/
# Repeat for decisions-review and best-practices-review as needed.
```

The skill then activates as `/guide-review` (un-namespaced).

## How a review works

1. You ask Claude to "review with Google Style Guide" (or `/google-go-styleguide:guide-review`, etc.).
2. The chosen skill identifies the `.go` files in scope.
3. For each `actionable` rule (or section), the skill **dispatches a subagent in parallel** with just that one rule file and the file list.
4. Each subagent loads only its own rule/section file, reads the code, and returns findings as `<file>:<line> — <observation>` or `no findings`.
5. The main agent groups results by rule and produces the final report.

This keeps the main context small (each rule file is loaded only inside its own subagent) and each check focused on a single principle. You can also ask for a partial review (e.g. "only check `#initialisms` and `#in-band-errors`") or a single-rule explanation.

## Skill structure

Each skill follows the same shape:

```
skills/<skill-name>/
├── SKILL.md             # frontmatter + index + review workflow
└── <content-dir>/       # principles/, guidelines/, or sections/
    └── NN-<slug>.md     # one file per rule (guide-review) or per section (decisions-review)
```

See each skill's `SKILL.md` for the full rule index and review instructions.

## Categories

Every rule is tagged with one of three categories:

| Category | Behaviour during review |
|---|---|
| `actionable` | Subagent generates per-line findings |
| `tooling` | Subagent verifies the tool is wired up (e.g. `gofmt` in CI), no per-line findings |
| `philosophical` | Skipped during review; informs judgement and is referenced in finding rationale |

## Sources

All rule content is synthesized from official Google documentation. Every rule file lists its primary sources at the bottom. Top-level sources used across the plugin:

- [Google Go Style Guide](https://google.github.io/styleguide/go/guide) — Skill 1 source
- [Google Go Style Decisions](https://google.github.io/styleguide/go/decisions) — Skill 2 source
- [Google Go Best Practices](https://google.github.io/styleguide/go/best-practices) — Skill 3 source
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Specification](https://go.dev/ref/spec)
- [Identifier naming (Google Testing Blog)](https://testing.googleblog.com/2017/10/code-health-identifiernamingpostforworl.html)
- [`gofumpt`](https://github.com/mvdan/gofumpt) — referenced in `guide-review`'s `#06 Formatting`
- [`go-cmp`](https://pkg.go.dev/github.com/google/go-cmp/cmp) — referenced in `decisions-review`'s `#types-of-equality`

## Related

- Sister plugin/skill: [`go-proverbs-review`](https://github.com/andrsj/go-proverbs-review-skill) — reviews against [Rob Pike's Go Proverbs](https://go-proverbs.github.io/). Different lens, complementary findings.

## License

MIT — see [LICENSE](LICENSE).
