---
id: P016
name: using-mise-action
date: 2026-07-14
author: Julio Capote
used_in: []
---

# Using mise-action in CI

`jdx/mise-action` is the GitHub Action that installs the tools declared in `mise.toml` straight into a CI runner, so `mise.toml` (P003) is the **single source of truth for tool versions in CI as well as locally** — not just a reference document that CI approximates with its own install steps. One step reads `mise.toml`, fetches every tool, and puts them on PATH for the rest of the job.

## When to use

- Any project that already follows P003 (mise) and wants CI to run the **exact** tool versions developers and agents run, with zero drift.
- Any project whose CI runs mise-managed tools that have no first-class `setup-*` action (oxlint/oxfmt (P014/P015), or any GitHub-release binary declared via `"github:owner/repo"`) — the action installs all of them from one config.
- Any project that wants to delete hand-rolled `curl | bash` install steps from its workflow and let `mise.toml` drive CI provisioning.

This is the CI half of P003. P003 allows that "CI often does not use mise — it has its own install mechanisms," with `mise.toml` as the reference. P016 is for when you close that gap and make CI use mise too, so there is no second install step to keep in sync.

## How it works

### One step installs everything mise.toml declares

Add the action after checkout (and after any package-manager setup). No inputs are required for the basic case — it reads `mise.toml` from the repo root, installs every declared tool, and adds them to PATH for subsequent steps:

```yaml title=".github/workflows/ci.yml"
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha> # v5
      - name: Setup mise
        uses: jdx/mise-action@<sha> # v4.2.0
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm exec tsc --noEmit
```

After this step, every tool in `mise.toml` — language runtimes, GitHub-release binaries, the oxc pair (P014/P015) — is on PATH. The later steps invoke them by bare name (`oxlint`, `oxfmt`, `actionlint`, `tsc`, …) exactly as a developer would locally. There is no per-tool `curl`, `apt-get`, or `setup-*` step to maintain.

### Pin the action by SHA with a version comment (P002)

Like every third-party action, `jdx/mise-action` is pinned to its full 40-character commit SHA with a version tag in a trailing comment, per P002:

```yaml title=".github/workflows/ci.yml"
- uses: jdx/mise-action@e6a8b3978addb5a52f2b4cd9d91eafa7f0ab959d # v4.2.0
```

The SHA is immutable; the `# v4.2.0` comment is the human-readable version. This is the same rule as pinning `actions/checkout` or `pnpm/action-setup`.

### The GITHUB_TOKEN is used automatically for GitHub-backend tools

Several `mise.toml` tools are GitHub-release binaries (`"github:owner/repo" = "vX.Y.Z"`, and the `[tool_alias]` github backends like oxlint/oxfmt). Downloading release assets counts against GitHub's anonymous rate limit (60 requests/hour per IP) unless authenticated. `mise-action` automatically uses the workflow's `GITHUB_TOKEN` for those GitHub-backend downloads, which lifts the limit.

Because the action reads the token from the job's default `GITHUB_TOKEN`, the job's permissions must allow it. Reading release assets is covered by the default read token — the standard job-level permission is enough:

```yaml title=".github/workflows/ci.yml"
permissions:
  contents: read
```

You do not need to pass a `token:` input or grant write permissions for plain tool downloads. (Add an explicit `token:` input only if the default token is unavailable in some step configuration.)

### mise.toml is the single source of truth — change it first

Because the action installs straight from `mise.toml`, there is **no CI-side version to update separately**. Bumping a tool is a one-file change:

1. Edit the version in `mise.toml`.
2. Run `mise install` locally to confirm it resolves.
3. Push. CI picks up the new version from the same line.

This is the strict version of P003's "update mise.toml first, then update the CI install step to match." With `mise-action`, the second half of that sentence disappears — there is no CI install step. The version cannot drift because there is only one place it's written.

```yaml title=".github/workflows/ci.yml"
# WRONG — installing a mise-managed tool by hand in CI alongside mise-action.
# This version is a second source of truth that will drift from mise.toml.
- name: Install actionlint
  run: |
    curl -fsSL https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash \
      | bash -s -- 1.7.12 /usr/local/bin
- name: Setup mise
  uses: jdx/mise-action@e6a8b3978addb5a52f2b4cd9d91eafa7f0ab959d # v4.2.0
```

If `actionlint` is already in `mise.toml`, the `curl` step above is dead weight: `mise-action` already installed it, and now the version is declared in two places. Delete the hand-rolled step and let `mise.toml` drive it.

### What mise-action does NOT replace

`mise-action` provisions the mise-managed tools. It does not replace:

- **Checkout** — still `actions/checkout@<sha>` (P002).
- **The package manager** — pnpm (P004) / npm / uv (P010) still need their own setup (`pnpm/action-setup`, `actions/setup-node` for the Node runtime if Node is *not* in `mise.toml`, `uv` if Python is not in `mise.toml`). If the language runtime IS in `mise.toml` (e.g. `node = "22"`), `mise-action` provides it and the corresponding `setup-*` action can be dropped; if it is not, keep the `setup-*` action. Pick one home for each runtime — do not declare `node` in both `mise.toml` and `setup-node`.
- **Install of project dependencies** — `pnpm install --frozen-lockfile` still installs `node_modules`; `mise-action` only provides the tools, not the packages.

### Trust is handled automatically

On a fresh clone, mise refuses an untrusted `mise.toml` interactively (P003). In CI this is not an obstacle: no separate trust step is needed — the action installs and runs against the checked-out config without prompting.

## The AGENTS.md section

```markdown title="AGENTS.md"
## CI tool versions

CI installs its tools with `jdx/mise-action`, which reads `mise.toml` and puts every declared tool on PATH for the job. There are no per-tool `curl` or `setup-*` steps for mise-managed tools.

- To change a tool version in CI, change it in `mise.toml` and push — there is no separate CI version to update.
- The action is SHA-pinned with a version comment, like every other action (P002).
- Do not add a hand-rolled install step for a tool that is already in `mise.toml` — it becomes a second source of truth that drifts.
```

## Anti-patterns to avoid

- **Installing a mise-managed tool by hand in CI (`curl | bash`, `apt-get`).** If the tool is in `mise.toml`, `mise-action` already installed it; a second install is a second version to keep in sync and will drift. Let `mise.toml` drive it.
- **Pinning `mise-action` by tag only (`@v4.2.0`).** Tags are mutable. Pin to the full SHA with the tag in a comment, like every other third-party action (P002).
- **Declaring a runtime in both `mise.toml` and a `setup-*` action.** If `node` is in `mise.toml`, `mise-action` provides it; drop `setup-node`. Two homes for one runtime is two versions to drift. Pick one per runtime.
- **Passing a `token:` input unnecessarily.** The action uses the job's default `GITHUB_TOKEN` for GitHub-backend downloads; the standard `permissions: contents: read` is enough. Only pass an explicit token if the default is unavailable.
- **Treating `mise.toml` as documentation while CI installs tools another way.** That is the looser posture P003 allows. If you adopt `mise-action`, commit to it: `mise.toml` is the source of truth, full stop, and CI has no independent version list.
- **Expecting `mise-action` to install project dependencies.** It installs tools, not packages. `pnpm install` / `npm ci` / `uv sync` still need to run.
- **Forgetting checkout before the action.** `mise-action` reads `mise.toml` from the checked-out repo; it must run after `actions/checkout`.
