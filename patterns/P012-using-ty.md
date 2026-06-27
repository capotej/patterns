---
id: P012
name: using-ty
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/hermes-harness-plugin']
---

# Using ty

ty is the Python type checker by Astral — a fast, mypy/pyright-alternative that's configured to run with zero project config by default. It's installed as a uv dev dependency (P010) and invoked with `uv run ty check <dir>` pointing at the source. It complements ruff (P011): ruff lints, ty type-checks. The convention is: declare it as a dev dep, run it with `uv run ty check src`, and gate it in CI.

## When to use

- Any Python project (P010) that wants type checking.

## How it works

### Declare ty as a uv dev dependency

ty is a Python package, so it goes in the dev dependencies alongside ruff (P011) and pytest — not in `mise.toml` (P003 is for non-Python tools). Per P010:

```toml title="pyproject.toml"
[project.optional-dependencies]
dev = ["pytest>=7", "ruff>=0.13", "ty>=0.0.1"]
```

ty is young (0.0.x). The exact version is resolved and pinned in the committed `uv.lock` (P010). When the version changes, edit `pyproject.toml` then `uv lock`.

### Zero config by default — point it at the source

Unlike ruff (P011), the reference repo commits **no `[tool.ty]` section**. ty is zero-config capable: it infers the Python version from `requires-python` and checks the directory you point it at. You run it against the package source, not the whole repo:

```bash
uv run ty check src    # check the source package
```

Pointing at `src` (the package dir) is deliberate — it type-checks the shipped code. Pointing at `.` would pull in tests, config, and scripts, which are often intentionally loose on types and would add noise.

If you do need to tune ty, configure it in `[tool.ty]` inside `pyproject.toml` (the P010 single-config-surface rule) — never a separate `ty.toml`. But the starting assumption is no config.

### Run with uv run

```bash
uv run ty check src        # type-check the package
```

Per P010, there is no bare `ty`/`python` on PATH (PEP 668) — always `uv run`.

### Complementary to ruff, not a replacement

ty and ruff (P011) cover different ground and are used together:

- **ruff** — linter (style, unused imports, bug-prone patterns, import sorting).
- **ty** — type checker (do the types actually line up).

ruff does not type-check; ty does not lint. A project following this library uses both, as two separate CI steps.

### Gate type-check in CI

ty runs as its own CI step, mirroring local development. It installs from the lock (P010), so no separate version is declared in CI:

```yaml title=".github/workflows/ci.yml"
- name: Install dependencies
  run: uv sync --locked --extra dev

- name: ty check
  run: uv run ty check src
```

The `setup-uv` action is SHA-pinned per P002. The version comes from `uv.lock`, so there's no version-sync problem — unlike the mise tools (P008, P009) where CI re-declares the version.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Type checking

This project type-checks Python with **ty**. It's a dev dependency (installed
via `uv sync --extra dev`) and runs with no project config.

- Type-check the package: `uv run ty check src`

ty complements ruff (linting) — it checks types, not style. Point it at `src`,
not the whole repo.
```

## Anti-patterns to avoid

- **Installing ty globally instead of as a dev dep.** The version is pinned in `uv.lock` (P010). A global install bypasses the lock and runs a different version than CI.
- **Pointing `ty check` at `.` instead of the source dir.** It drags in tests and config, which are often deliberately un-typed, and floods the output with noise. Point at the package (`src`).
- **Not gating in CI.** A type checker that only runs locally is advisory. The gate is what makes it a convention.
- **Configuring ty in a separate `ty.toml`.** If you need config, it goes in `[tool.ty]` in `pyproject.toml` per P010. But the default is no config — don't add one speculatively.
- **Running ty alongside mypy/pyright over the same code.** ty is a type checker; running a second type checker over the same source produces redundant, conflicting findings. Pick one. (ty and ruff are fine together — they check different things.)

## Reference

### boldblackai/hermes-harness-plugin

```toml title="pyproject.toml"
[project.optional-dependencies]
dev = ["pytest>=7", "ruff>=0.13", "ty>=0.0.1"]
```

```yaml title=".github/workflows/ci.yml"
- name: ty check
  run: uv run ty check src
```

```markdown title="AGENTS.md (excerpt)"
- use `ty` for type checking (`uv run ty check`)
```

A Hermes Agent plugin. Declares ty as a dev dep (`ty>=0.0.1`, exact pin in `uv.lock`); runs it with zero config — no `[tool.ty]` section, just `uv run ty check src` pointed at the source package; gates it as its own CI step alongside ruff (P011) and pytest. ty complements ruff here: ruff lints, ty type-checks.
