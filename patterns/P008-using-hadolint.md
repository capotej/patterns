---
id: P008
name: using-hadolint
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/harness']
---

# Using hadolint

hadolint is the Dockerfile linter — a standalone binary provisioned by mise (P003), NOT a pnpm devDependency, because it's a Haskell tool rather than a Node package. It has nothing to do with pnpm; it's a CLI you run directly. The convention is: provision it via mise, configure it via a committed `.hadolint.yaml`, and gate it in CI. How the project invokes the tool (a shell script, a make target, a package.json script, or a direct call) is the project's choice.

## When to use

- Any project that ships a Dockerfile and wants it linted in CI.
- Any project already following P003 (mise) — hadolint installs alongside the other non-Node CLI tools (shellcheck, actionlint) in `mise.toml`.

## How it works

### Provision hadolint via mise, not pnpm

hadolint is a Haskell binary with no Node dependency, so it does NOT go in `package.json` devDependencies. It's declared in `mise.toml` as a GitHub-sourced tool (P003), pinned to an exact release tag:

```toml title="mise.toml"
[tools]
"github:hadolint/hadolint" = "v2.14.0"
```

This is the key contrast with Biome (P005) and markdownlint (P007): those are npm packages installed via pnpm; hadolint is a standalone binary installed via mise. Use the `github:owner/repo` source because hadolint isn't in mise's built-in registry. Pin to the exact release tag including the `v` prefix.

Take the latest stable hadolint release and pin that.

### Commit a .hadolint.yaml

hadolint is zero-config capable, but the reference repo commits an explicit `.hadolint.yaml` at the repo root. The committed file sets the lint gate:

```yaml title=".hadolint.yaml"
failure-threshold: error
```

`failure-threshold: error` means hadolint exits non-zero only on rule violations classified as `error` severity; `warning` and `info` are reported but don't fail the build. This is the intentional setting when you want signal without noise — many hadolint rules (e.g. pinning apt versions) are warnings by default and would be too noisy as hard failures.

Other common keys (not all used by the reference repo, but the standard shape):

```yaml title=".hadolint.yaml"
# Optional: disable specific rules by ID
ignored:
  - DL3008   # pin apt versions

# Optional: trust specific registries
trustedRegistries:
  - ghcr.io/boldblackai
```

### Invocation

hadolint is a plain CLI: `hadolint <Dockerfile>...`. It takes a list of files as arguments — there's no recursive globbing, so every Dockerfile must be named explicitly:

```bash
hadolint Dockerfile Dockerfile.opencode Dockerfile.hermes
```

When you add a new Dockerfile, add it to whatever invocation the project uses (script, make target, CI step). A Dockerfile not listed is silently un-linted.

### Gate lint in CI

hadolint should run in CI. The version installed in CI MUST match the version in `mise.toml` — mise is the source of truth (P003); CI follows it. When the version changes, update `mise.toml` first, then the CI install step.

CI installs hadolint directly (not via mise) using the release binary:

```yaml title=".github/workflows/lint.yml"
- name: Install system linters
  run: |
    curl -fsSL -o /usr/local/bin/hadolint \
      https://github.com/hadolint/hadolint/releases/download/v2.14.0/hadolint-Linux-x86_64
    chmod +x /usr/local/bin/hadolint
- run: hadolint Dockerfile Dockerfile.opencode Dockerfile.hermes
```

The download URL embeds the version (`v2.14.0`) which must match mise.toml. GitHub Actions are SHA-pinned per P002 where third-party actions are used.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Dockerfile Linting

This project lints Dockerfiles with **hadolint**. The tool is installed via mise (`"github:hadolint/hadolint" = "v2.14.0"` in `mise.toml`), not via pnpm — activate mise before running it.

Configuration lives in `.hadolint.yaml` (failure threshold set to `error`). When you add a new Dockerfile, add its filename to the lint invocation — hadolint does not glob. When changing the hadolint version, update `mise.toml` first and the CI install step second.
```

## Anti-patterns to avoid

- **Installing hadolint via pnpm or as a global npm package.** hadolint is a Haskell binary, not a Node package. It belongs in `mise.toml` as a GitHub-sourced tool (P003). An npm wrapper adds a needless Node dependency layer.
- **Pinning hadolint in CI but not in mise.toml.** mise is the source of truth for tool versions (P003). If the curl URL in CI says `v2.14.0` and mise.toml says something else, they've drifted. Update mise.toml first, CI second.
- **Forgetting to add a new Dockerfile to the lint invocation.** hadolint takes an explicit file list — there's no glob. A new Dockerfile not in that list is silently un-linted.
- **Setting `failure-threshold` to `info` or `warning`.** Many hadolint rules are warnings by default (e.g. DL3008 pin apt versions). Lowering the threshold to warnings turns the lint into a wall of noise; raising it to `error` keeps the signal-to-noise ratio intentional.
- **Linting a Dockerfile that's not checked in.** hadolint lints source Dockerfiles. If you generate Dockerfiles at build time (e.g. via `docker buildx` bake), the generated output shouldn't be linted.

## Reference

### boldblackai/harness

```toml title="mise.toml"
[tools]
"github:hadolint/hadolint" = "v2.14.0"
```

```yaml title=".hadolint.yaml"
failure-threshold: error
```

A containerized agent runner. Provisions hadolint as a mise tool pinned to `v2.14.0`; configures a single `failure-threshold: error` in `.hadolint.yaml`. harness wires its linters as pnpm scripts and runs them via a single `pnpm lint` command — `lint:docker` is `hadolint Dockerfile Dockerfile.opencode Dockerfile.hermes` — but that orchestration is a harness choice, not part of the hadolint convention; the tool itself is a standalone CLI with no dependency on pnpm. The CI install step downloads the same `v2.14.0` declared in mise.toml, so the two stay in sync.
