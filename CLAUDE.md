# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is — and what it is not yet

**refactoring-cli** is an AgentCulture mesh agent whose intended domain is:
analyse a Python codebase for structural problems and code smells, propose
behaviour-preserving refactorings with a rationale, and apply them under test.
It is the successor to the retired `refactor-cli`, whose atomic-transformation
lane it absorbs.

**None of that domain logic exists yet.** The tree is still the
`culture-agent-template` scaffold (`git log`: two commits, the second being
`scaffold refactoring-cli from culture-agent-template`). What ships today is the
agent-first CLI skeleton — `whoami`, `learn`, `explain`, `overview`, `doctor`,
`cli overview` — plus a mesh identity, a vendored skill kit, and a full CI/deploy
baseline. Several docstrings and catalog entries still describe the repo as "a
clonable template for AgentCulture mesh agents"; that prose is inherited, not a
statement about this agent's purpose.

Treat the scaffold as the *contract* a future `refactor`/`analyse`/`apply` noun
group must satisfy, not as throwaway code — the `teken` rubric gate in CI enforces
its shape.

## Commands

```bash
uv sync                                   # install (dev group included)

# Tests
uv run pytest                             # full suite
uv run pytest -n auto                     # parallel (what CI runs)
uv run pytest tests/test_cli.py::test_whoami_json -v          # a single test
uv run pytest -k "doctor" -v                                   # by name
uv run pytest -n auto --cov=refactor --cov-report=term         # with coverage
bash .claude/skills/run-tests/scripts/test.sh --ci              # exact CI invocation

# Lint — CI runs all five, all must pass
uv run black --check refactor tests
uv run isort --check-only refactor tests
uv run flake8 refactor tests
uv run bandit -c pyproject.toml -r refactor
markdownlint-cli2 "**/*.md" "#node_modules" "#.local" "#.claude/skills"

# The agent-first rubric gate (CI job `lint` → "afi rubric gate")
uv run teken cli doctor . --strict

# Run the CLI
uv run refactor whoami            # note: `refactor`, not `refactoring-cli` — see below
uv run python -m refactor learn --json
```

`black` and `isort` are checked but not auto-run — apply fixes with
`uv run black refactor tests && uv run isort refactor tests`. Line length is 100
everywhere (`.flake8`, `[tool.black]`, `[tool.isort]` agree).

Coverage has `fail_under = 60` in `pyproject.toml`, so `--cov` runs fail below
that threshold. `relative_files = true` is load-bearing: absolute paths in
`coverage.xml` don't map to `sonar.sources=refactor` and SonarCloud reports empty
coverage.

## Known inconsistency: `refactor` vs `refactoring-cli`

The scaffold rename is incomplete. Today:

| Thing | Value |
|-------|-------|
| Python package directory | `refactor/` |
| Console script (`[project.scripts]`) | `refactor` |
| Distribution / project name | `refactoring-cli` |
| argparse `prog=`, all help text, `explain` catalog, tests | `refactoring-cli` |

So `uv run refactoring-cli whoami` — as `README.md` instructs — **fails**; the
working invocation is `uv run refactor whoami` or `python -m refactor`. The
rubric gate passes because `teken` discovers the console script name from
`pyproject.toml`. `whoami` reports `nick: refactoring-cli` because the nick comes
from `culture.yaml`, not from the package name.

Before adding domain code, decide which name wins and finish the rename. Discover
every occurrence first — the name is hard-coded in ~100 places:

```bash
git grep -n -E 'refactoring[-_]cli|refactoring_cli' -- . ':!uv.lock' ':!.claude/skills'
git grep -n -w -E 'refactor' -- refactor tests pyproject.toml sonar-project.properties
```

Targets: `pyproject.toml` (`name`, `[project.scripts]`, `[tool.hatch...packages]`,
`[tool.coverage.run] source`, `[tool.isort] known_first_party`, `[tool.bandit]`),
the package directory itself, every import in `refactor/` and `tests/`,
`_ISSUES_URL` in `refactor/cli/__init__.py`, `sonar-project.properties`
(`sonar.projectKey` must match the registered SonarCloud project), `README.md`,
and the `refactor/explain/catalog.py` `ENTRIES` keys (which currently register
*both* `("refactoring-cli",)` and `("refactor",)` as root aliases).

## Architecture

Three layers, each with a contract the rubric gate checks:

**1. Entry + dispatch — `refactor/cli/__init__.py`**

`main(argv)` → `_build_parser()` → `parse_args` → `_dispatch(args)`. Two things
here are subtle and easy to break:

- `_CliArgumentParser` overrides `argparse.ArgumentParser.error()` so *parse-time*
  failures (unknown verb, bad flag) render through the same `error:` / `hint:`
  contract as runtime failures and exit `1`, not argparse's default exit `2`.
  Subparsers must be created with `parser_class=_CliArgumentParser` or they fall
  back to argparse's behaviour and the rubric's `error_has_hint` check fails. The
  `cli` noun group does this explicitly via `parser_class=type(p)` — copy that
  pattern for any nested noun.
- `_json_hint` is a class attribute set by scanning raw argv *before* parsing,
  because `args.json` doesn't exist yet when a parse error fires. That is how
  `--json` is honoured for argparse-level errors.
- `_dispatch` wraps *any* non-`CliError` exception into a `CliError`, so no Python
  traceback ever reaches stderr (rubric: `no_traceback`).

**2. Commands — `refactor/cli/_commands/*.py`**

Every module exposes `register(sub: argparse._SubParsersAction) -> None` and is
wired in `_build_parser()`. Handlers return `int | None` (None ⇒ 0) and raise
`CliError` on failure — never `sys.exit`, never print to stderr directly.

- `whoami.py` owns identity resolution and is imported by `doctor.py` and
  `overview.py`. `find_culture_yaml()` walks up from `__file__`, **not** the cwd —
  the agent's identity must be its own, not whatever `culture.yaml` sits in the
  caller's directory. In a wheel install no `culture.yaml` ships and everything
  falls back to literals.
- `read_agent_fields()` hand-parses `culture.yaml` line by line. This is
  deliberate: `[project] dependencies = []` — the runtime package has **zero**
  third-party dependencies, so there is no PyYAML to lean on. Keep it that way;
  new deps belong in `[dependency-groups] dev`.
- `doctor.py` mirrors the two invariants `steward doctor` checks —
  `prompt_file_present` and backend-consistency — via the `_PROMPT_FILE` map
  (`claude`→`CLAUDE.md`, `colleague`→`AGENTS.colleague.md`, `acp`→`AGENTS.md`,
  `gemini`→`GEMINI.md`), plus `skills_present`. It emits the rubric-shaped
  `{healthy, checks: [{id, passed, severity, message, remediation}]}`.
- `overview.py` holds the shared section/render helpers that `cli.py` reuses.
  Descriptive verbs must never hard-fail on a bad target path — hence `overview`'s
  ignored optional `target` positional (rubric:
  `overview_graceful_on_bad_path`).

**3. Explain catalog — `refactor/explain/`**

`ENTRIES: dict[tuple[str, ...], str]` maps command-path tuples to verbatim
markdown; `resolve()` raises `CliError` with a hint on a miss. Bodies are meant
to be self-contained — an agent reading one entry shouldn't need to chain reads.
`tests/test_cli.py::test_every_catalog_path_resolves` walks every key.

**The output contract — `refactor/cli/_output.py`**

Results to **stdout**, errors and diagnostics to **stderr**, never mixed, in both
text and JSON mode. Text errors are exactly two lines (`error:` then `hint:`); the
`hint:` prefix is what the rubric greps for. Exit codes live in `_errors.py`:
`0` success, `1` user error, `2` environment error, `3+` reserved. This contract
is a stable API for agent consumers — the docstrings say `stable-contract` for a
reason.

### Adding a noun group or verb

The rubric and tests span several files, so a new verb touches all of these:

1. New module in `refactor/cli/_commands/` with `register(sub)`; add
   `--json` to every parser you create.
2. Register it in `_build_parser()` (there's a marked comment block for it).
3. Add a `refactor/explain/catalog.py` entry — and one for each sub-verb path.
4. Add it to `_TEXT` **and** `_as_json_payload()["commands"]` in `learn.py`.
5. Add it to `_VERBS` in `overview.py` (feeds both `overview` and `cli overview`).
6. Tests in `tests/` covering text mode, `--json` shape, and the error path.
7. If the noun takes action-verbs, it must also expose `<noun> overview` — the
   rubric's `overview_cli_noun_exists` check. `_commands/cli.py` exists solely as
   the reference implementation of that pattern.
8. Re-run `uv run teken cli doctor . --strict`.

## Conventions and workflow

**Every PR bumps the version — including docs-only, config-only, and CI-only
changes.** The `version-check` CI job compares `pyproject.toml` against
`origin/main` and fails (with a PR comment) when they match. Use the skill:

```bash
echo '{"added":["..."],"changed":["..."],"fixed":["..."]}' \
  | python3 .claude/skills/version-bump/scripts/bump.py minor
```

It updates `pyproject.toml` and prepends a Keep-a-Changelog entry in one step.

**PR lane.** Use the `cicd` skill (`.claude/skills/cicd/scripts/workflow.sh`),
which delegates `lint | open | read | reply | delta` to `devex pr` and adds
`status` (SonarCloud gate + hotspots + unresolved-thread tally) and `await`
(poll then gate, non-zero on Sonar `ERROR` or unresolved threads). Requires
`devex >= 0.21`, `gh`, `jq`, `curl` on PATH. Don't hand-sign PR replies — the
scripts resolve the nick from `culture.yaml` and append `- refactoring-cli
(Claude)` themselves. For issue bodies and PR descriptions the scripts don't
author, sign manually as `- refactoring-cli (Claude)`.

**Vendored skills are cited, not owned.** Everything under `.claude/skills/` is
copied verbatim from `guildmaster`, `devague`, or `colleague` — **do not edit
them**. `docs/skill-sources.md` is the provenance ledger, including three tracked
local divergences (`agex`→`devex` in-place patch; `ask-colleague` vendored direct
from `colleague`; `scope`/`challenge`/`deviate`/`summarize-delivery` vendored
direct from `devague` to avoid guildmaster's added wrapper scripts). Follow its
re-sync procedure when pulling updates, and lift wanted changes upstream rather
than patching locally. Both `markdownlint` and Sonar exclude the directory.
Note that every vendored `SKILL.md` must carry `type: command` in its frontmatter
— `core.skill_loader` silently skips any that lacks it.

**Worktrees** you create by hand (workforce fan-out lanes, scratch checkouts) go
in `../.worktrees.refactoring-cli/<name>/` — one repo-named directory beside the
checkout, so a stale-tree sweep across this multi-project workspace can tell
which trees belong to which repo. Scope branch prefixes to the work at hand; a
plain `agent/*` prefix collides with leftovers from earlier fan-outs and fails
`git worktree add -b`. Tear down with `git worktree remove <path>` — `prune` only
clears metadata for directories already gone. The vendored `assign-to-workforce`
skill's example uses a shared `../worktrees/` path and `agent/<task-id>`
branches; it's cited verbatim and must not be edited, so override both when
following it. `ask-colleague`'s read-only verbs make their own throwaway
worktrees under `$TMPDIR` and reap them on exit — out of scope.

**Memory.** Recall before, remember after. Run `/recall` before non-trivial work
so prior decisions get built on rather than re-derived, and `/remember` when a
non-obvious decision, constraint, fix-and-why, or hard-won gotcha surfaces. The
vendored wrappers default to this agent's personal private scope (`--scope
refactoring-cli --visibility private`, store `~/.eidetic/memory`); pass
`--visibility public` to contribute to the shared pool. Requires `eidetic` on
PATH.

**Second opinions.** Reach for `ask-colleague` reflexively, not only when asked:
`review` before presenting or opening a PR on a non-trivial committed diff, and
`explore` for a fresh read of an unfamiliar area. Both are read-only and isolated
in a throwaway worktree, so they're always safe. The side-effecting
`write --apply` / `write --pr` needs the user's go-ahead first.

**Local config.** Copy `.claude/skills.local.yaml.example` →
`.claude/skills.local.yaml` (git-ignored) to point skills at the Culture server
manifest and sibling checkouts.

## Identity and deploy

`culture.yaml` declares one agent: `suffix: refactoring-cli`, `backend:
colleague`, with a pinned model. Because the backend is `colleague`, the
load-bearing resident prompt file is **`AGENTS.colleague.md`**, not this file —
`doctor`'s backend-consistency check will fail if it goes missing. Changing
`backend` means changing the prompt file to match *and* teaching `doctor`'s
`_PROMPT_FILE` map; `test_doctor_recognizes_declared_backend` guards exactly that.

`.github/workflows/publish.yml` publishes to PyPI via Trusted Publishing on push
to `main` (env `pypi`) and to TestPyPI on same-repo PRs (env `testpypi`,
version suffixed `.devN`). Both are path-filtered on `pyproject.toml` and
`refactor/**`, and fork PRs skip publishing. SonarCloud gating in `tests.yml` is
guarded by `env.SONAR_TOKEN != ''`, so token-less and fork runs stay green.
