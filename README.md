# refactoring-cli

Refactors Python code. Analyses a Python codebase for structural problems and code smells, proposes behaviour-preserving refactorings with a rationale, and applies them under test. Successor to the retired refactor-cli, whose atomic-transformation lane it absorbs.

## Status

**The refactoring lane is not implemented yet.** This repo currently carries the
scaffold it was minted from: an agent-first CLI skeleton (identity and
introspection verbs), a mesh identity, the vendored skill kit, and a working
build/test/deploy baseline. The analyse-propose-apply surface described above is
the target, not today's behaviour.

What that scaffold buys is a contract for the domain verbs to land inside —
`--json` on every command, a strict stdout/stderr split, structured errors with
remediation hints, and a documented exit-code policy — all enforced in CI by the
agent-first rubric gate.

## What's here

- **An agent-first CLI** cited from [teken](https://github.com/agentculture/teken)
  (`afi-cli`) — the runtime package has no third-party dependencies.
- **A mesh identity** — `culture.yaml` (`suffix` + `backend`) and the matching
  resident prompt file (`AGENTS.colleague.md`, since this agent runs
  `backend: colleague`).
- **The vendored skill kit** (18 skills) under `.claude/skills/`, cited
  cite-don't-import from guildmaster, devague, and colleague. See
  [`docs/skill-sources.md`](docs/skill-sources.md) for provenance and the
  re-sync procedure.
- **A build + deploy baseline** — pytest, lint, the agent-first rubric gate, and
  PyPI Trusted Publishing wired into GitHub Actions.

## Quickstart

```bash
uv sync
uv run pytest -n auto                 # run the test suite
uv run refactor whoami                # identity from culture.yaml
uv run refactor learn                 # self-teaching prompt (add --json)
uv run teken cli doctor . --strict    # the agent-first rubric gate CI runs
```

## CLI

The console script is **`refactor`**; `refactoring-cli` is the distribution name
on PyPI. `python -m refactor` works too.

| Verb | What it does |
|------|--------------|
| `whoami` | Report this agent's nick, version, backend, and model from `culture.yaml`. |
| `learn` | Print a structured self-teaching prompt. |
| `explain <path>` | Markdown docs for any noun/verb path. |
| `overview` | Read-only descriptive snapshot of the agent. |
| `doctor` | Check the agent-identity invariants (prompt-file-present, backend-consistency). |
| `cli overview` | Describe the CLI surface itself. |

Every command supports `--json`. Results go to stdout, errors/diagnostics to
stderr (never mixed). Exit codes: `0` success, `1` user error, `2` environment
error, `3+` reserved.

### Naming

The rename from the template this repo was scaffolded from is incomplete: the
package directory and console script are `refactor`, while the distribution
name, argparse `prog`, help text, `explain` catalog, and tests all say
`refactoring-cli`. Finish the rename in one pass before the domain verbs land —
[`CLAUDE.md`](CLAUDE.md) carries the discovery command and the full target list.

## Contributing

Every PR bumps the version, including docs-only and CI-only changes — a
`version-check` job blocks the merge otherwise. See [`CLAUDE.md`](CLAUDE.md) for
the full conventions (the `cicd` PR lane, the vendored-skills rule, worktree
layout, deploy setup).

## License

Apache 2.0 — see [`LICENSE`](LICENSE).
