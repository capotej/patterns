---
id: P002
name: absolutely-pinned-github-actions
date: 2026-06-21
author: Julio Capote
used_in: ['github.com/boldblackai/harness', 'github.com/boldblackai/hermes-harness-plugin']
---

# Absolutely Pinned GitHub Actions

Every third-party action reference in a workflow file must be pinned to its full 40-character commit SHA, with the corresponding version tag in a trailing comment. No exceptions.

## When to use

- Any repository that runs GitHub Actions workflows and wants supply-chain integrity guarantees.
- Any project where a compromised action tag could lead to stolen secrets, poisoned artifacts, or CI backdoors.

## How it works

### The rule

Every `uses:` line that points to a third-party action must follow this form:

```yaml
uses: owner/repo@<40-char-commit-sha> # <version-tag>
```

Concretely:

```yaml
# Correct
uses: actions/checkout@93cb6efe18208431cddfb8368fd83d5badbf9bfd # v5

# Wrong — tag-only (mutable, can be force-pushed)
uses: actions/checkout@v5

# Wrong — SHA without tag comment (unreadable, can't tell what version)
uses: actions/checkout@93cb6efe18208431cddfb8368fd83d5badbf9bfd

# Wrong — short SHA (not unique enough, not verifiable)
uses: actions/checkout@93cb6efe
```

### Why SHA-pin?

Git tags are mutable. Anyone with push access to an action repository can force-push a `v5` tag to point at a malicious commit. Your CI would happily run the attacker's code with `secrets.GITHUB_TOKEN` and any custom secrets in scope.

Commit SHAs are immutable — the same SHA always refers to the same tree. Pinning to a SHA means the code your CI runs is the code you audited, even if the tag is later moved.

### Why the tag comment?

A raw SHA is a 40-character hex string that tells a reader nothing about what version is in use. The trailing `# v5` or `# v6.8.0` comment restores readability for humans reviewing or upgrading workflows, without affecting execution.

### Tag comment format

The tag comment should match the version string the action's maintainer publishes. Both of these forms are acceptable:

```yaml
uses: actions/checkout@93cb6efe18208431cddfb8368fd83d5badbf9bfd # v5
uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
```

- Major-only (`# v5`) is common and fine.
- Full semver (`# v6.0.2`) is more precise and preferred when you know the exact version.

### Local composite actions

Local composite actions (e.g. `./.github/actions/attest-provenance`) are exempt from SHA-pinning because they resolve within the same checkout. They are already pinned by the repository's own commit.

```yaml
# Local composite action — no SHA needed
uses: ./.github/actions/attest-provenance
```

### Reusable workflows

Reusable workflows called with `uses: owner/repo/.github/workflows/flow.yml@ref` follow the same rule — pin `@ref` to a SHA with a tag comment:

```yaml
uses: boldblackai/harness/.github/workflows/docker.yml@93cb6efe18208431cddfb8368fd83d5badbf9bfd # v5
```

## How to resolve a tag to a SHA

Using the GitHub CLI:

```bash
gh api repos/actions/checkout/commits/v5 --jq '.sha'
```

Or via the Git protocol:

```bash
git ls-remote https://github.com/actions/checkout.git refs/tags/v5
```

## How to enforce

### With pinact

[pinact](https://github.com/suzuki-shunsuke/pinact) automatically pins unpinned actions and updates existing pins:

```bash
# Pin all actions in a repo
pinact run

# Check for unpinned actions in CI
pinact run --check
```

### With actionlint

[actionlint](https://github.com/rhysd/actionlint) can warn about unpinned actions with its built-in checks. Add to CI:

```yaml
- name: Lint workflows
  run: actionlint -ignore 'shellcheck reported issue'
```

### In AGENTS.md

Add a section to your agent instructions to enforce the convention:

```markdown title="AGENTS.md"
## GitHub Actions

All third-party action references in `.github/workflows/` must be pinned to their full 40-character commit SHA with a version tag in a trailing comment:

    uses: owner/repo@<sha> # <tag>

Never use tag-only references (e.g. `actions/checkout@v5`). When adding or updating an action, resolve the tag to a SHA using `gh api repos/OWNER/REPO/commits/TAG --jq '.sha'`. Local composite actions (`./.github/actions/*`) are exempt.
```

## Reference: actions used across Bold Black AI repos

| Action | SHA | Tag |
|---|---|---|
| `actions/checkout` | `93cb6efe18208431cddfb8368fd83d5badbf9bfd` | `v5` |
| `actions/checkout` | `de0fac2e4500dabe0009e67214ff5f5447ce83dd` | `v6.0.2` |
| `actions/setup-node` | `48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e` | `v6` |
| `actions/upload-artifact` | `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` | `v7` |
| `actions/download-artifact` | `3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c` | `v8` |
| `actions/configure-pages` | `45bfe0192ca1faeb007ade9deae92b16b8254a0d` | `v6` |
| `actions/upload-pages-artifact` | `fc324d3547104276b827a68afc52ff2a11cc49c9` | `v5` |
| `actions/deploy-pages` | `cd2ce8fcbc39b97be8ca5fce6e763baed58fa128` | `v5` |
| `docker/setup-buildx-action` | `d7f5e7f509e45cec5c76c4d5afdd7de93d0b3df5` | `v4` |
| `docker/login-action` | `650006c6eb7dba73a995cc03b0b2d7f5ca915bee` | `v4` |
| `docker/metadata-action` | `80c7e94dd9b9319bd5eb7a0e0fe9291e23a2a2e9` | `v6` |
| `docker/build-push-action` | `f9f3042f7e2789586610d6e8b85c8f03e5195baf` | `v7` |
| `sigstore/cosign-installer` | `398d4b0eeef1380460a10c8013a76f728fb906ac` | `v3` |
| `astral-sh/setup-uv` | `08807647e7069bb48b6ef5acd8ec9567f424441b` | `v8.1.0` |
| `astral-sh/setup-uv` | `d0cc045d04ccac9d8b7881df0226f9e82c39688e` | `v6.8.0` |
| `pnpm/action-setup` | `0e279bb959325dab635dd2c09392533439d90093` | `v6` |
