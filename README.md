# Patterns

This repo houses a collection of common patterns that can be mixed and matched together.

## Pattern Index

| ID | Pattern | Description |
| --- | --- | --- |
| [P001](patterns/P001-rfc-process.md) | RFC Process | Structured proposal workflow for non-trivial changes. |
| [P002](patterns/P002-absolutely-pinned-github-actions.md) | Absolutely Pinned GitHub Actions | Pin GitHub Actions by immutable SHA with a `# tag` comment. |
| [P003](patterns/P003-using-mise.md) | Using mise | `mise.toml` as the single source of truth for tool/language versions. |
| [P004](patterns/P004-using-pnpm.md) | Using pnpm | pnpm declared via `packageManager`, committed lockfile, gitignored store. |
| [P005](patterns/P005-using-biome.md) | Using Biome | Biome lint/format pinned with a version-matched `biome.json`. |
| [P006](patterns/P006-using-typescript.md) | Using TypeScript | `tsc` with strict mode, single `src/` root, declaration emit. |
| [P007](patterns/P007-using-markdownlint.md) | Using markdownlint-cli2 | Markdown lint via `.markdownlint-cli2.jsonc`, composed into `pnpm lint`. |
| [P008](patterns/P008-using-hadolint.md) | Using hadolint | Dockerfile lint via mise-provisioned binary + `.hadolint.yaml`, gated in CI. |
| [P009](patterns/P009-using-actionlint.md) | Using actionlint | GitHub Actions workflow lint via mise binary + `.actionlint.yaml`, gated in CI. |
| [P010](patterns/P010-using-uv-and-pyproject-toml.md) | Using uv and pyproject.toml | `pyproject.toml` as the single config source, committed `uv.lock`, `uv run` everywhere (PEP 668). |
| [P011](patterns/P011-using-ruff.md) | Using ruff | Python linter as a uv dev dep, configured in `[tool.ruff]`, `uv run ruff check .`, gated in CI. |
| [P012](patterns/P012-using-ty.md) | Using ty | Python type checker as a uv dev dep, zero-config, `uv run ty check src`, gated in CI. |
| [P013](patterns/P013-code-notes.md) | Code Notes (GHC-style "Notes") | Titled, cross-referenceable block comments (`Note [Title]` / `See Note [Title]`) that describe the present state, not the journey. |

## Pattern Format

### Location & Filename

Patterns are markdown files that live inside of `patterns/`, named sequentially like: `patterns/P001-name-of-the-pattern.md`.

### Frontmatter Metadata

Patterns must contain the following YAML frontmatter at the beginning of their file:

```yaml
---
id: P001
name: name-of-the-pattern
date: 2026-07-15
author: John Doe
used_in: ['github.com/capotej/example-repo']
---
```

### Structure

Patterns contain fenced markdown codeblocks (always containing language and filename) surrounded by explanations of how/when they should be used.

Example:

```typescript title="userService.ts"
export class UserService {
}
```

## Usage

Point your agent at this repository with a prompt like "Implement patterns P031 and P021 in this project".


