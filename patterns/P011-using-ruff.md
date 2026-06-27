---
id: P011
name: using-ruff
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/hermes-harness-plugin']
---

# Using ruff

ruff is the Python linter (and formatter) — a Rust-based tool by Astral that replaces flake8, isort, pyupgrade, and friends in one binary. It's installed as a uv dev dependency (P010) and configured entirely in `[tool.ruff]` inside `pyproject.toml` — never a separate `ruff.toml`. The convention is: declare it as a dev dep, configure it in pyproject, run it with `uv run ruff check .`, and gate it in CI.

## When to use

- Any Python project (P010) that wants a fast, unified linter.

## How it works

### Declare ruff as a uv dev dependency

ruff is a Python package, so it goes in the dev dependencies — not in `mise.toml` (P003 is for non-Python tools). Per P010, dev deps are declared once in `pyproject.toml` and installed via the `dev` extra (or `dependency-groups`):

```toml title="pyproject.toml"
[project.optional-dependencies]
dev = ["pytest>=7", "ruff>=0.13", "ty>=0.0.1"]
```

The exact version is resolved and pinned in the committed `uv.lock` (P010). When the version changes, edit `pyproject.toml` then `uv lock`.

### Configure in pyproject.toml, not a separate file

All ruff config lives in `[tool.ruff]` inside `pyproject.toml` — this is the P010 single-config-surface rule. Do not create a `ruff.toml` or `.ruff.toml` alongside it; a second config file is a second source of truth that drifts.

```toml title="pyproject.toml"
[tool.ruff]
line-length = 99
target-version = "py313"

[tool.ruff.lint]
# Keep it sensible: pyflakes + pycodestyle + import sorting + useful extras.
select = ["E", "F", "I", "UP", "B", "SIM"]
```

Two settings matter:

1. **`target-version`** — MUST match the project's Python version. It tells ruff which language features are valid (so `UP`/pyupgrade rules suggest the right modern syntax). If `requires-python = ">=3.13"`, then `target-version = "py313"`. A mismatch is a bug.
2. **`line-length`** — per-project; the convention is that it's declared once here, not that it's any specific value. ruff's default is 88; the reference repo uses 99.

Rule selection uses `select` with rule prefixes. Each letter group is a tool ruff replaces:

| Prefix | What it checks |
|---|---|
| `E`, `W` | pycodestyle (style/whitespace) |
| `F` | Pyflakes (unused imports, undefined names) |
| `I` | isort (import sorting) |
| `UP` | pyupgrade (modernize syntax for target-version) |
| `B` | flake8-bugbear (common pitfalls) |
| `SIM` | flake8-simplify (simplifiable code) |

The point is one `select` list replacing half a dozen flake8 plugins. Disable a specific rule with `ignore = ["E501"]` (with a comment explaining why).

### Run with uv run

```bash
uv run ruff check .        # lint the whole project
uv run ruff check src      # lint a directory
uv run ruff check --fix .  # auto-fix what's auto-fixable
```

Per P010, there is no bare `ruff`/`python` on PATH (PEP 668) — always `uv run`.

ruff also ships a formatter (`ruff format`, a black replacement). The reference repo gates only `ruff check`; running `uv run ruff format` for formatting is a complementary, optional step — not part of the lint gate.

### Gate lint in CI

ruff runs as its own CI step, mirroring local development. It installs from the lock (P010), so no separate version is declared in CI:

```yaml title=".github/workflows/ci.yml"
- name: Install dependencies
  run: uv sync --locked --extra dev

- name: Ruff check
  run: uv run ruff check .
```

The `setup-uv` action is SHA-pinned per P002. Because the version comes from `uv.lock`, there's no version-sync problem to manage — unlike the mise tools (P008, P009) where CI re-declares the version.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Linting

This project lints Python with **ruff**. It's a dev dependency (installed via
`uv sync --extra dev`) and configured in `[tool.ruff]` in `pyproject.toml`.

- Lint: `uv run ruff check .`
- Auto-fix: `uv run ruff check --fix .`

Do not create a separate `ruff.toml`. Keep `target-version` in sync with
`requires-python`.
```

## Anti-patterns to avoid

- **Configuring ruff in a separate `ruff.toml`.** Per P010, all tool config lives in `pyproject.toml`'s `[tool.ruff]`. A standalone file drifts from it.
- **`target-version` not matching `requires-python`.** ruff uses it to decide valid syntax. `requires-python = ">=3.13"` with `target-version = "py38"` suppresses valid modernization suggestions and can flag correct code.
- **Running ruff alongside the tools it replaces.** ruff bundles flake8, isort, pyupgrade, bugbear, etc. Keeping a parallel flake8/isort config (or pre-commit hook) is redundant and the two will disagree. Pick ruff.
- **Installing ruff globally instead of as a dev dep.** The version is pinned in `uv.lock` (P010). A global install bypasses the lock and runs a different version than CI.
- **Not gating in CI.** A linter that only runs locally is advisory. The gate is what makes it a convention.
- **Ignoring a rule without a comment.** Like the markdownlint/actionlint ignores (P007, P009), an `ignore` entry should carry an inline rationale.

## Reference

### boldblackai/hermes-harness-plugin

```toml title="pyproject.toml"
[project.optional-dependencies]
dev = ["pytest>=7", "ruff>=0.13", "ty>=0.0.1"]

[tool.ruff]
line-length = 99
target-version = "py313"

[tool.ruff.lint]
# Keep it sensible: pyflakes + pycodestyle + import sorting + useful extras.
select = ["E", "F", "I", "UP", "B", "SIM"]
```

```yaml title=".github/workflows/ci.yml"
- name: Ruff check
  run: uv run ruff check .
```

```markdown title="AGENTS.md (excerpt)"
- use `ruff` for linting (`uv run ruff check`)
```

A Hermes Agent plugin. Declares ruff as a dev dep (`ruff>=0.13`, exact pin in `uv.lock`); configures it in `[tool.ruff]` with `line-length = 99` (per-project) and `target-version = "py313"` matching `requires-python = ">=3.13"`; selects a sensible rule set (`E`, `F`, `I`, `UP`, `B`, `SIM`); gates `ruff check .` as its own CI step. The formatter (`ruff format`) is not gated — ruff is used purely as a linter here.
