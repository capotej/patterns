---
id: P010
name: using-uv-and-pyproject-toml
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/hermes-harness-plugin']
---

# Using uv and pyproject.toml

A single `pyproject.toml` is the source of truth for everything — project metadata, runtime + dev dependencies, and every tool's config — with a committed `uv.lock` pinning the reproducible resolution, and `uv run` as the only way anything executes (there is no bare `python`/`pip` on PATH; PEP 668). uv manages the environment; `pyproject.toml` declares it.

## When to use

- Any Python project.

## How it works

### pyproject.toml is the single source of truth

All project metadata, dependencies, and tool config live in `pyproject.toml`. There is no `requirements.txt`. Dependencies go under `[project].dependencies`:

```toml title="pyproject.toml"
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "my-project"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "rich>=13.0.0",
    "aiofiles>=24.0.0",
]
```

uv is backend-agnostic — it works with any PEP 517 build backend (setuptools, hatchling, flit). The choice of backend is a packaging concern, not a uv concern.

**`pyproject.toml` is the single config surface for the whole project.** Runtime deps, dev deps, the build backend, *and* every tool's configuration all live in it — uv's own `[tool.uv]`, the linter `[tool.ruff]`, the test runner `[tool.pytest.ini_options]`, setuptools `[tool.setuptools.*]`. Do not scatter separate `ruff.toml`, `pytest.ini`, `setup.cfg`, or `setup.py` files alongside it; they become second sources of truth that drift.

```toml title="pyproject.toml — tool config also lives here"
[tool.ruff]
line-length = 99
target-version = "py313"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-q"

[tool.uv]
exclude-newer = "7d"
```

### The uv.lock file (committed)

`uv.lock` is committed to the repo. It records the exact, fully-resolved dependency graph (including transitive deps) with hashes, making installs reproducible across machines and CI. It replaces `requirements.txt` and `poetry.lock`/`Pipfile.lock`.

- **Never hand-edit `uv.lock`.** It is generated.
- **Re-generate it after changing dependencies** in `pyproject.toml` with `uv lock`, then commit the result.
- **The lock must stay in sync with pyproject.toml.** Editing deps without re-locking leaves a stale lock that will drift on the next `uv sync`.

### uv run is the universal entry point

This is the core rule. Everything runs through `uv run`, which executes the command inside the project's venv with the locked dependencies:

```bash
uv run pytest              # run tests
uv run python script.py    # run a script
uv run ruff check .        # run the linter
uv run ty check src        # run the type checker
uv run mypackage           # run a console script declared in [project.scripts]
```

**Why:** modern environments (and every containerized agent image) are PEP 668 — there is no bare `python` or `pip` on PATH, and `pip install` into the system interpreter is blocked. `uv run` is the escape hatch: it creates/manages the project venv automatically and never touches the system interpreter. Running `python ...` or `pip ...` directly fails or corrupts the environment.

`uv run` also auto-syncs the environment first if needed, so a fresh clone needs no separate `uv sync` before the first command.

### Installing / refreshing the environment

```bash
uv sync          # create the venv and install everything in the lock
```

`uv sync` is what you run after cloning or pulling dependency changes. `uv run` calls it implicitly when the environment is out of date.

### Python version

The Python interpreter version is declared, not assumed. There are two cooperating sources:

1. **`requires-python`** in `pyproject.toml` — the supported range. This is required and authoritative.
2. **`.python-version`** (optional) — pins the exact interpreter for local development. uv reads it and will download the matching managed interpreter if missing.

uv can also manage interpreters directly: `uv python install 3.13` installs a managed Python 3.13 (useful in CI where the runner's default may differ). Whichever you use, the version must agree across all sources — see the anti-pattern on version conflicts.

### Dev dependencies: two valid styles (per-project choice)

Dev-only tools (pytest, ruff, ty) are declared separately from runtime deps. There are two equally valid uv-native styles — **pick one per project, do not mix**:

```toml title="pyproject.toml — PEP 735 dependency-groups (modern, preferred)"
[dependency-groups]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.13",
    "ty>=0.0.19",
]
# install with:  uv sync --group dev
```

```toml title="pyproject.toml — optional-dependencies extra (classic)"
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.13",
]
# install with:  uv sync --extra dev
```

Both work. Use `dependency-groups` (PEP 735) for a pure-dev toolchain that never ships as installable extras. Use `optional-dependencies` when you want consumers to install the extras (e.g. `pip install my-project[dev]`) or need broad pip/older-toolchain compatibility. The convention is *that dev deps are declared once in pyproject and installed via a flag* — which mechanism you use is per-project.

### Reproducibility hardening (recommended)

A committed, hashed `uv.lock` already makes the *resolved* graph reproducible. Two complementary knobs make the *resolution* itself deterministic rather than "whatever is newest at resolve time":

```toml title="pyproject.toml"
[tool.uv]
# Rolling cutoff: ignore any distribution published in the last 7 days.
# Re-evaluated on every `uv lock`, so it never goes stale. Pairs with the
# exact pins (incl. hashes) in uv.lock to make resolution reproducible.
exclude-newer = "7d"
```

```bash
uv sync --locked     # fail if uv.lock is out of date with pyproject.toml
```

`--locked` is the important one for CI: it makes a stale lock fail loudly instead of silently re-resolving to different versions. `exclude-newer` adds a supply-chain safety margin by holding back brand-new releases.

### CI

CI installs uv via the `astral-sh/setup-uv` action, syncs from the lock, and runs commands with `uv run` — exactly mirroring local development:

```yaml title=".github/workflows/ci.yml"
- name: Install uv
  uses: astral-sh/setup-uv@<pinned-sha> # v6.8.0   # per P002, pin by SHA
  with:
    enable-cache: true

- name: Set up Python
  run: uv python install 3.13

- name: Install dependencies
  run: uv sync --locked --extra dev     # --group dev if using dependency-groups

- name: Test
  run: uv run pytest
```

CI does not install Python or dependencies via `apt-get`/`pip`/`setup-python` — it uses uv, so the lock is the source of truth there too. If a dependency version changes, update `pyproject.toml`, run `uv lock`, commit, and CI picks it up. Per P002, pin the `setup-uv` action itself by SHA with a `# tag` comment.

## The AGENTS.md section

Add this to your agent instructions so coding agents know to use uv:

```markdown title="AGENTS.md"
## Python toolchain (uv)

This project uses **uv**. Do not use bare `python` or `pip` — there is none on
PATH (PEP 668). `pyproject.toml` is the source of truth for metadata and
dependencies; `uv.lock` is the committed, reproducible resolution.

- Run anything in the project venv: `uv run <command>`
  (e.g. `uv run pytest`, `uv run python script.py`).
- Install/refresh the env from the lock: `uv sync` (add `--group dev` or
  `--extra dev` to include dev deps — match whichever the project uses).
- Add a dependency by editing `pyproject.toml`, then `uv lock` to re-resolve
  and commit the updated `uv.lock`. Never hand-edit `uv.lock`.
- In CI, install with `uv sync --locked` so a stale lock fails instead of
  silently re-resolving.

There is no `requirements.txt`.
```

## Anti-patterns to avoid

- **Running `python` or `pip` directly.** On PEP 668 systems there is no bare interpreter and system installs are blocked; use `uv run python ...` / `uv run pip ...` instead. The AGENTS.md should state this explicitly so agents don't guess.
- **Hand-editing `uv.lock`.** It is generated. Change deps in `pyproject.toml`, then `uv lock`.
- **Changing dependencies without re-locking.** Editing `[project].dependencies` but forgetting `uv lock` leaves the lock stale; `uv sync` will silently re-resolve to different versions, destroying reproducibility.
- **Omitting `--locked` in CI.** Without it, CI happily re-resolves a stale lock instead of failing. A stale lock in CI is a bug that `--locked` turns into a loud failure.
- **Keeping a `requirements.txt`.** The committed `uv.lock` is the dependency record; a parallel `requirements.txt` is a second source of truth that will drift. Delete it.
- **Declaring the Python version in two places that disagree.** `requires-python`, `.python-version`, and (if you also follow P003) `mise.toml` must all agree on the Python version — a mismatch is a bug, not a fallback.
- **Mixing dependency-groups and optional-dependencies for the same dev toolchain.** Pick one style per project. Having both `dependency-groups.dev` and `optional-dependencies.dev` is confusing and risks installing different sets.
- **Splitting config into `setup.py` / `setup.cfg` / `requirements-dev.txt`.** Everything lives in `pyproject.toml`. Legacy config files are how versions silently diverge.

## Reference

### boldblackai/hermes-harness-plugin

The canonical best-practice example — every recommendation in this pattern is present here (single `pyproject.toml` config surface, committed hashed `uv.lock`, `uv run` everywhere, `--locked` CI gating, `exclude-newer`, SHA-pinned actions per P002).

```toml title="pyproject.toml"
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[project]
name = "hermes-harness-plugin"
dynamic = ["version"]
requires-python = ">=3.13"
dependencies = [ ... ]

[project.optional-dependencies]
dev = ["pytest>=7", "ruff>=0.13", "ty>=0.0.1"]

[tool.setuptools.dynamic]
version = { attr = "hermes_harness_plugin.__version__" }

[tool.uv]
exclude-newer = "7d"
```

```yaml title=".github/workflows/ci.yml"
- name: Install uv
  uses: astral-sh/setup-uv@d0cc045d04ccac9d8b7881df0226f9e82c39688e # v6.8.0
  with:
    enable-cache: true

- name: Set up Python
  run: uv python install 3.13

- name: Install dependencies
  run: uv sync --locked --extra dev

- name: Ruff check
  run: uv run ruff check .
- name: ty check
  run: uv run ty check src
- name: Test
  run: uv run pytest
```

```markdown title="AGENTS.md (excerpt)"
## Toolchain
- **uv** is the dev tool. No `requirements.txt`; lock is `uv.lock`.
- There is no bare `python`/`pip` on PATH; the environment is PEP 668 (use `uv`).
  Run Python via `uv run python ...`.
```

Uses the classic `[project.optional-dependencies]` style (installed via `uv sync --extra dev`). Adds the `[tool.uv] exclude-newer = "7d"` rolling cutoff and gates CI with `uv sync --locked`, and SHA-pins both `setup-uv` and `checkout` (per P002). No `.python-version`; CI installs Python explicitly with `uv python install 3.13`, consistent with `requires-python = ">=3.13"`.
