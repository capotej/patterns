---
id: P008
name: using-hadolint
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/harness']
---

# Using hadolint

hadolint is the Dockerfile linter — a standalone binary provisioned by mise (P003), NOT a pnpm devDependency, because it's a Go tool rather than a Node package. It's configured via a committed `.hadolint.yaml`, exposed as `lint:docker`, and composed into the single `pnpm lint`. hadolint is the Dockerfile arm that Biome (P005) and markdownlint (P007) don't cover.

## When to use

- Any project that ships a Dockerfile and wants it linted in CI.
- Any project already following P003 (mise) — hadolint installs alongside the other non-Node CLI tools (shellcheck, actionlint) in `mise.toml`.
- Any project where `pnpm lint` should fan out to Dockerfiles too, even though hadolint itself isn't a Node tool.

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

### The lint:docker script and the composed lint

hadolint is exposed as its own script and composed into `lint`:

```json title="package.json"
{
  "scripts": {
    "lint": "pnpm lint:ts && pnpm lint:md && pnpm lint:sh && pnpm lint:docker && pnpm lint:actions",
    "lint:docker": "hadolint Dockerfile Dockerfile.opencode Dockerfile.hermes"
  }
}
```

- `lint:docker` — invokes `hadolint` with every Dockerfile in the repo as explicit arguments. hadolint takes a list of files; there's no recursive globbing, so every Dockerfile must be named. When you add a new Dockerfile, add it to this script.
- `lint` — the single entry point that runs every linter in sequence. `lint:docker` is one arm, alongside `lint:ts` (Biome, P005), `lint:md` (markdownlint, P007), `lint:sh` (shellcheck), and `lint:actions` (actionlint).

The rule: there is ONE command — `pnpm lint` — and it fans out to every file type. Developers and agents never need to know the per-type script names.

### Gate lint in CI

hadolint runs in CI in one of two ways:

1. **Via the lint workflow** (where CI installs mise and runs `pnpm lint`). The agent's shell activates mise, so `hadolint` resolves to the pinned binary.
2. **Via a manual install in the lint job** (where CI installs hadolint directly rather than using mise):

```yaml title=".github/workflows/lint.yml"
steps:
  - uses: actions/checkout@<sha> # v5
  - uses: pnpm/action-setup@<sha> # v6
  - uses: actions/setup-node@<sha> # v6
    with:
      node-version: 24
      cache: pnpm
  - run: pnpm install --frozen-lockfile
  - name: Install system linters
    run: |
      sudo apt-get update && sudo apt-get install -y shellcheck
      curl -fsSL -o /usr/local/bin/hadolint \
        https://github.com/hadolint/hadolint/releases/download/v2.14.0/hadolint-Linux-x86_64
      chmod +x /usr/local/bin/hadolint
  - run: pnpm lint
```

Key point: the version hadolint is installed at in CI (`v2.14.0` in the curl URL) MUST match the version in `mise.toml`. mise is the source of truth (P003); CI follows it. When the version changes, update `mise.toml` first, then the CI install step.

The GitHub Actions are SHA-pinned per P002; pnpm setup follows P004.

### Relationship to other linters

hadolint is part of a five-linter stack orchestrated by `pnpm lint`:

| Script | Tool | Scope | Provisioned via |
| --- | --- | --- | --- |
| `lint:ts` | Biome | JS/TS | pnpm devDep (P005) |
| `lint:md` | markdownlint-cli2 | Markdown | pnpm devDep (P007) |
| `lint:sh` | shellcheck | Shell scripts | mise (P003) |
| `lint:docker` | hadolint | Dockerfiles | mise (P003) |
| `lint:actions` | actionlint | GitHub Actions YAML | mise (P003) |

The split is clean by file type: Biome owns compiled code, markdownlint owns docs, and the three non-Node linters (shellcheck/hadolint/actionlint) own their respective config/script files. None of them overlap.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Dockerfile Linting

This project lints Dockerfiles with **hadolint**. The tool is installed via mise (`"github:hadolint/hadolint" = "v2.14.0"` in `mise.toml`), not via pnpm — activate mise before running it.

- Lint all Dockerfiles: `pnpm lint:docker`
- Lint everything (Dockerfiles + all other file types): `pnpm lint`

Configuration lives in `.hadolint.yaml` (failure threshold set to `error`). When you add a new Dockerfile, add its filename to the `lint:docker` script — hadolint does not glob. When changing the hadolint version, update `mise.toml` first and the CI install step second.
```

## Anti-patterns to avoid

- **Installing hadolint via pnpm or as a global npm package.** hadolint is a Haskell binary, not a Node package. It belongs in `mise.toml` as a GitHub-sourced tool (P003). An npm wrapper adds a needless Node dependency layer.
- **Pinning hadolint in CI but not in mise.toml.** mise is the source of truth for tool versions (P003). If the curl URL in CI says `v2.14.0` and mise.toml says something else, they've drifted. Update mise.toml first, CI second.
- **Forgetting to add a new Dockerfile to the lint:docker script.** `hadolint Dockerfile Dockerfile.opencode Dockerfile.hermes` lists files explicitly — there's no glob. A new Dockerfile not in that list is silently un-linted.
- **Setting `failure-threshold` to `info` or `warning`.** Many hadolint rules are warnings by default (e.g. DL3008 pin apt versions). Lowering the threshold to warnings turns the lint into a wall of noise; raising it to `error` keeps the signal-to-noise ratio intentional.
- **Linting a Dockerfile that's not checked in.** hadolint lints source Dockerfiles. If you generate Dockerfiles at build time (e.g. via `docker buildx` bake), the generated output shouldn't be linted.
- **Running hadolint outside the composed `pnpm lint`.** The convention is a single `pnpm lint` that fans out to every file type. A separate ad-hoc `hadolint` invocation bypasses the rest of the lint pass.
- **Letting hadolint and Biome overlap.** hadolint owns Dockerfiles; Biome (P005) owns JS/TS. Don't add a second Dockerfile linter or try to make Biome lint Dockerfiles.

## Reference

### boldblackai/harness

```toml title="mise.toml"
[tools]
"github:hadolint/hadolint" = "v2.14.0"
```

```yaml title=".hadolint.yaml"
failure-threshold: error
```

```json title="package.json"
{
  "scripts": {
    "lint": "pnpm lint:ts && pnpm lint:md && pnpm lint:sh && pnpm lint:docker && pnpm lint:actions",
    "lint:docker": "hadolint Dockerfile Dockerfile.opencode Dockerfile.hermes"
  }
}
```

```yaml title=".github/workflows/lint.yml"
- name: Install system linters
  run: |
    sudo apt-get update && sudo apt-get install -y shellcheck
    curl -fsSL -o /usr/local/bin/hadolint \
      https://github.com/hadolint/hadolint/releases/download/v2.14.0/hadolint-Linux-x86_64
    chmod +x /usr/local/bin/hadolint
- run: pnpm lint
```

A containerized agent runner. Provisions hadolint as a mise tool pinned to `v2.14.0`; configures a single `failure-threshold: error` in `.hadolint.yaml`; lists all three Dockerfiles (`Dockerfile`, `Dockerfile.opencode`, `Dockerfile.hermes`) explicitly in `lint:docker`; composes that into the five-part `pnpm lint` and gates it in `lint.yml`. The CI install step downloads the same `v2.14.0` declared in mise.toml, so the two stay in sync.
