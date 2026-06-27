---
id: P009
name: using-actionlint
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/harness']
---

# Using actionlint

actionlint is the GitHub Actions workflow linter — a standalone Go binary provisioned by mise (P003), NOT a pnpm devDependency. It has nothing to do with pnpm; it's a CLI you run directly. The convention is: provision it via mise, configure it via a committed `.actionlint.yaml`, and gate it in CI. How the project invokes the tool is the project's choice.

## When to use

- Any project with `.github/workflows/*.yml` that wants workflow YAML linted in CI.
- Any project already following P003 (mise) — actionlint installs alongside the other non-Node CLI tools (shellcheck, hadolint) in `mise.toml`.

## How it works

### Provision actionlint via mise, not pnpm

actionlint is a Go binary with no Node dependency, so it does NOT go in `package.json` devDependencies. It's declared in `mise.toml` as a GitHub-sourced tool (P003), pinned to an exact release tag:

```toml title="mise.toml"
[tools]
"github:rhysd/actionlint" = "v1.7.7"
```

This mirrors hadolint (P008): both are standalone binaries installed via mise, not npm packages installed via pnpm. Use the `github:owner/repo` source because actionlint isn't in mise's built-in registry. Pin to the exact release tag including the `v` prefix.

Take the latest stable actionlint release and pin that.

### Commit a .actionlint.yaml

actionlint is zero-config capable, but the reference repo commits an explicit `.actionlint.yaml` at the repo root. The committed file holds the `ignore` list — and every ignore carries an inline comment explaining why:

```yaml title=".actionlint.yaml"
ignore:
  # docker.yml uses common GH Actions patterns (>> $GITHUB_ENV) that trigger shellcheck
  - 'shellcheck reported issue'
```

The philosophy is identical to markdownlint's `.markdownlint-cli2.jsonc` (P007): exceptions are allowed, but each one must be justified by a comment naming the concrete reason. An ignore without a rationale is a gap.

The `'shellcheck reported issue'` string matches a class of findings that actionlint produces when its internal shellcheck pass flags a workflow `run:` block. More on that below.

### Invocation (auto-discovers workflows)

actionlint is a plain CLI. Unlike hadolint (P008), it takes NO file arguments — it auto-discovers `.github/workflows/*.yml` (and `*.yaml`, and composite actions referenced under `.github/actions/`). You run it bare:

```bash
actionlint
```

Adding a new workflow file is automatically covered. Don't add explicit file arguments or a glob — a glob miss (`*.yml` when you meant `*.yaml`) would silently lint nothing.

### Built-in shellcheck integration

actionlint doesn't just lint YAML structure — it runs shellcheck on the shell inside every `run:` block and on shell scripts referenced by workflows. This is a deliberate overlap with standalone shellcheck usage on repo scripts:

```yaml title=".github/workflows/docker.yml"
- name: Extract short SHA
  run: echo "SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)" >> "$GITHUB_ENV"
```

Patterns like `>> "$GITHUB_ENV"` are idiomatic GitHub Actions shell but trip shellcheck's generic rules. The `.actionlint.yaml` ignore silences exactly that class (`'shellcheck reported issue'`) so the workflow lint stays useful rather than drowning in false positives from correct GH Actions idioms.

So shellcheck runs in two places: standalone on repo shell scripts, and embedded inside actionlint on workflow `run:` blocks. The two don't conflict — they cover different files.

### Gate lint in CI

actionlint should run in CI. The version installed in CI MUST match the version in `mise.toml` — mise is the source of truth (P003); CI follows it. actionlint's recommended install path pipes the official download script and passes the pinned version as an argument:

```yaml title=".github/workflows/lint.yml"
- name: Install actionlint
  run: |
    curl -fsSL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash \
      | bash -s -- 1.7.7 /usr/local/bin
- run: actionlint
```

Key points:

- The version passed to the download script (`1.7.7`) MUST match the version in `mise.toml`. Note the format quirk: mise uses the release tag with `v` prefix (`v1.7.7`); the download script takes the bare version (`1.7.7`).
- When the version changes, update `mise.toml` first, then the CI install step.
- GitHub Actions are SHA-pinned per P002 where third-party actions are used.

## The AGENTS.md section

```markdown title="AGENTS.md"
## GitHub Actions Linting

This project lints `.github/workflows/*.yml` with **actionlint**. The tool is installed via mise (`"github:rhysd/actionlint" = "v1.7.7"` in `mise.toml`), not via pnpm — activate mise before running it. actionlint also runs shellcheck on every `run:` block, so workflow shell gets checked even though it's not in the repo's shell scripts.

Configuration lives in `.actionlint.yaml`. New workflow files are discovered automatically — no file arguments needed. When changing the actionlint version, update `mise.toml` first and the CI install step second.
```

## Anti-patterns to avoid

- **Installing actionlint via pnpm or as a global npm package.** actionlint is a Go binary, not a Node package. It belongs in `mise.toml` as a GitHub-sourced tool (P003). Mirrors the P008 rule for hadolint.
- **Pinning actionlint in CI but not in mise.toml (or vice versa).** mise is the source of truth for tool versions (P003). The `1.7.7` passed to the CI download script must match `v1.7.7` in mise.toml. When it changes, update mise.toml first, CI second. Watch the format: mise tag has the `v`, the download script arg doesn't.
- **Adding explicit file arguments to the invocation.** Unlike hadolint (P008), actionlint auto-discovers `.github/workflows/`. Adding `actionlint .github/workflows/*.yml` is redundant and brittle (a glob miss silently lints nothing). Run it bare.
- **Ignoring without a comment.** The `.actionlint.yaml` ignore list uses YAML comments to justify each entry, exactly like markdownlint's JSONC comments (P007). An ignore with no rationale is an unexplained hole.
- **Suppressing the embedded shellcheck pass wholesale.** actionlint's internal shellcheck catches real bugs in `run:` blocks. The reference repo ignores only the `'shellcheck reported issue'` class — patterns that are correct GH Actions idioms but trip generic shellcheck. Don't expand the ignore to silence real findings.
- **Introducing a second workflow linter.** actionlint owns `.github/workflows/`. Don't add super-linter, pre-commit hooks, or a custom YAML validator that duplicates it.

## Reference

### boldblackai/harness

```toml title="mise.toml"
[tools]
"github:rhysd/actionlint" = "v1.7.7"
```

```yaml title=".actionlint.yaml"
ignore:
  # docker.yml uses common GH Actions patterns (>> $GITHUB_ENV) that trigger shellcheck
  - 'shellcheck reported issue'
```

A containerized agent runner. Provisions actionlint as a mise tool pinned to `v1.7.7`; commits a `.actionlint.yaml` with a single ignore class (`shellcheck reported issue`) justified by a comment; invokes `actionlint` bare (auto-discovers `.github/workflows/`). harness wires its linters as pnpm scripts and runs them via a single `pnpm lint` command — `lint:actions` is just `actionlint` — but that orchestration is a harness choice, not part of the actionlint convention; the tool itself is a standalone CLI with no dependency on pnpm. The CI download script receives `1.7.7` — bare, matching mise's `v1.7.7` tag.
