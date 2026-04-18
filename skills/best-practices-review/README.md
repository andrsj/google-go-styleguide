# best-practices-review

Reviews Go code against [Google's Go Best Practices](https://google.github.io/styleguide/go/best-practices) — the **auxiliary, non-canonical** document of pragmatic guidance ("may not apply in every circumstance"). Findings are **advisory** ("should") rather than prescriptive ("must").

This is one of three skills in the [`google-go-styleguide`](../../README.md) Claude Code plugin. It can be installed and used standalone, or as part of the full plugin alongside [`guide-review`](../guide-review/) and [`decisions-review`](../decisions-review/).

## What it covers

12 sections covering ~50+ individual recommendations. Same per-section dispatch shape as `decisions-review`:

| # | Section | Sub-rules |
|---|---|---|
| 01 | Naming | 4 (function/method names, test double helpers, shadowing, util packages) |
| 02 | Package size | philosophical (no per-line findings) |
| 03 | Imports | 2 (protobuf/gRPC stub naming, deferral to Decisions on ordering) |
| 04 | Error handling | 7 (structure, adding info, `%w` placement, logging, program init, checks/panics, when to panic) |
| 05 | Documentation | 4 (conventions, preview, godoc formatting, signal boosting) |
| 06 | Variable declarations | 5 (initialization, zero values, composite literals, size hints, channel direction) |
| 07 | Function argument lists | 2 (option structure, variadic options) |
| 08 | Complex command-line interfaces | 1 (cobra vs subcommands; `cmd.Context()` pitfall) |
| 09 | Tests | 8 (leave to `Test`, extensible validation APIs, real transports, `Error` vs `Fatal`, helpers, no `Fatal` from goroutines, field names, scoped setup with `TestMain` / `sync.Once`) |
| 10 | String concatenation | 4 (`+`, `fmt.Sprintf`, `strings.Builder`, constant strings) |
| 11 | Global state | 3 (major forms of package state APIs, litmus tests, providing a default instance) |
| 12 | Interfaces | 3 (avoid unnecessary interfaces, ownership and visibility, designing effective interfaces) |

See [`SKILL.md`](SKILL.md) for the full review workflow and per-section index.

## Findings are advisory

The Best Practices doc explicitly opens with: *"This guidance is intended for common situations encountered while authoring Go in the Google codebase. It may not apply in every circumstance."*

So reports from this skill should phrase findings as recommendations, not violations. A finding here can be discussed and overruled by project context — that's expected. If the same code is also violating `guide-review` or `decisions-review`, those skills' findings take precedence.

## Install standalone

If you only want this one skill:

```bash
git clone https://github.com/andrsj/google-go-styleguide.git
cp -R google-go-styleguide/skills/best-practices-review ~/.claude/skills/
```

Activates as `/best-practices-review` (un-namespaced).

## Install as part of the plugin

See the [top-level README](../../README.md) for `--plugin-dir` instructions. When loaded as a plugin, this skill is invoked as `/google-go-styleguide:best-practices-review`.

## Precedence

Best Practices is the **lowest-precedence** document of Google's three. Where it conflicts with the [Style Guide](../guide-review/) or [Style Decisions](../decisions-review/), those win. The full hierarchy:

> Style Guide > Style Decisions > Best Practices

Install all three skills for comprehensive Google-style review.

## License

MIT — see the [top-level LICENSE](../../LICENSE).
