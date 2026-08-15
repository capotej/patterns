---
id: B001
name: typescript-project
date: 2026-08-15
author: Julio Capote
patterns: ['P002', 'P003', 'P004', 'P006', 'P007', 'P009', 'P014', 'P015', 'P016', 'P017']
used_in: ['github.com/capotej/tools']
---

# TypeScript Project

The baseline pattern set for a new TypeScript repository: a mise-pinned toolchain, pnpm with a committed lockfile, strict `tsc`, the oxc lint/format pair, markdownlint, SHA-pinned GitHub Actions workflows provisioned by mise-action, and the release skill. These are the ten patterns that keep landing together — implementing them as a bundle takes a repo from empty to fully-tooled in one pass, with every version pinned and every gate wired into CI.

Proven as a complete set in [capotej/tools](https://github.com/capotej/tools): the CLI was bootstrapped from exactly this combination (P002–P016 landed in one bootstrap commit; P017 was ported shortly after as its release runbook).

## When to use

- Starting a new TypeScript project from scratch — the bundle is the default opening move, not a menu to re-derive each time.
- Tooling up an existing TypeScript repo — the same set, applied incrementally.
- Any TypeScript project that will be released through an agent-driven pipeline.

## The patterns

| Order | Pattern | Role in the bundle |
| --- | --- | --- |
| 1 | [P003](../patterns/P003-using-mise.md) — Using mise | `mise.toml` as the single source of truth for tool versions. Everything else hangs off this. |
| 2 | [P004](../patterns/P004-using-pnpm.md) — Using pnpm | pnpm via `packageManager`, committed lockfile, exact-pinned deps. |
| 3 | [P006](../patterns/P006-using-typescript.md) — Using TypeScript | Strict `tsc`, NodeNext ESM, declaration emit. The compiler. |
| 4 | [P015](../patterns/P015-using-oxlint.md) — Using oxlint | JS/TS linter; correctness/suspicious/perf as hard errors. |
| 5 | [P014](../patterns/P014-using-oxfmt.md) — Using oxfmt | JS/TS/JSON formatter; `pnpm format` / `format:check`. |
| 6 | [P007](../patterns/P007-using-markdownlint.md) — Using markdownlint-cli2 | Markdown lint, composed into `pnpm lint`. |
| 7 | [P002](../patterns/P002-absolutely-pinned-github-actions.md) — Absolutely Pinned GitHub Actions | Every workflow action pinned by full SHA with a version comment. |
| 8 | [P009](../patterns/P009-using-actionlint.md) — Using actionlint | Workflow linting (plus shellcheck on every `run:` block). |
| 9 | [P016](../patterns/P016-using-mise-action.md) — Using mise-action in CI | CI installs the `mise.toml` tools — the CI half of P003. |
| 10 | [P017](../patterns/P017-release-skill.md) — The Release Skill | Agent-driven release runbook; lands last, once there is something to release. |

### Why this order

- **P003 first**: `mise.toml` pins Node (and actionlint); every later tool resolves through it.
- **P004 + P006 next**: package manager and compiler — at this point the project builds.
- **P015 / P014 / P007**: the lint and format gates, composed into `pnpm lint` so CI has a single entry point.
- **P002 → P009 → P016**: CI in three moves — pin the actions, lint the workflows, then let mise-action provision the runner from the same `mise.toml`.
- **P017 last**: the release skill wraps everything upstream and assumes the gates already exist.

## Deliberately excluded

- **P005 (Biome)** — superseded here by the oxc pair (P014/P015); running two lint stacks is noise.
- **P008 (hadolint)** — no Dockerfile in the baseline; add it when a container arrives.
- **P018 (Tokenless npm Publish)** — an orthogonal publishing concern; add it alongside P017 for packages published to npm via OIDC.

## Usage

Point your agent at this repository with a prompt like "Implement bundle B001 in this project". The agent resolves the bundle to its pattern list and implements each pattern in the order above.
