---
id: P018
name: tokenless-npm-publish
date: 2026-08-14
author: Hermclaw
used_in: ['github.com/capotej/tools']
---

# Tokenless npm Publish (OIDC Trusted Publishing)

Publishing npm packages traditionally requires a long-lived `NPM_TOKEN` secret — a secret that, once leaked, gives anyone publish access to your package until you rotate it. npm's OIDC trusted publishing eliminates that secret entirely: GitHub's OIDC provider asserts "this workflow, running on this repo, triggered by this tag push, is who it claims to be," and npm accepts that assertion instead of a token. No secret to leak, no rotation to remember, no `~/.npmrc` on developer machines.

The setup is a one-time configuration on npmjs.com (register the repo + workflow as a trusted publisher) and a GitHub Actions workflow that requests an `id-token` permission. The npm CLI auto-detects the OIDC environment and exchanges the token — no extra flags needed.

## When to use

- Any npm package published from GitHub Actions.
- Any repo where you want to eliminate long-lived npm secrets.
- Public packages (provenance is generated automatically; no `--provenance` flag needed).
- Repos that already use GitHub Actions for CI — the release workflow is just one more workflow file.

## How it works

### One-time npmjs.com configuration

On the package's npmjs.com settings page (or via the npm CLI), register a trusted publisher:

1. **Repository owner:** the GitHub owner (e.g. `capotej`).
2. **Repository name:** the repo name (e.g. `tools`).
3. **Workflow filename:** the exact filename of the publishing workflow (e.g. `release.yml`).
4. **Environment:** leave blank unless you use GitHub environments.

This tells npm: *"only accept publish requests from this workflow file in this repo, triggered by a tag push."* No npm token is generated or stored.

### The release workflow

The workflow triggers on version tag pushes and publishes with OIDC:

```yaml title=".github/workflows/release.yml"
name: Release

# Publishes to npm using OIDC trusted publishing — no long-lived npm token.
# Requires a trusted publisher configured on npmjs.com pointing at this
# workflow file (release.yml) for <owner>/<repo>. See AGENTS.md "Releases".
on:
  push:
    tags:
      - "v*"

permissions:
  id-token: write # OIDC token generation for trusted publishing
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09 # v5

      - name: Setup pnpm
        uses: pnpm/action-setup@f520eceda224fe1a4aed5a2a27a194379a409996 # v6

      - name: Setup Node
        uses: actions/setup-node@249970729cb0ef3589644e2896645e5dc5ba9c38 # v6
        with:
          node-version: 24
          cache: pnpm
          registry-url: "https://registry.npmjs.org"

      - run: pnpm install --frozen-lockfile
      - run: pnpm build

      # npm CLI auto-detects the OIDC environment and exchanges the token.
      # Provenance is generated automatically for public packages from
      # public repos — no --provenance flag needed.
      - run: npm publish
```

Key details:

- **`permissions.id-token: write`** — this is the only non-default permission needed. It lets GitHub generate an OIDC token that npm verifies.
- **`permissions.contents: read`** — checkout only; the workflow never writes back to the repo.
- **`registry-url`** on the Node setup step — this tells `npm publish` which registry to hit. Combined with OIDC, no `NODE_AUTH_TOKEN` env var is needed.
- **`npm publish`** with no extra flags — the npm CLI detects it's running in a GitHub Actions environment with `id-token` permission and automatically performs the OIDC exchange.
- **No `NPM_TOKEN` secret** — this is the whole point. There is nothing in `repository secrets` to leak or rotate.

### Pinned Actions

All Actions are pinned by immutable SHA (per [P002](patterns/P002-absolutely-pinned-github-actions.md)) with a `# tag` comment for readability. Never pin by tag or branch alone — a compromised or renamed tag could inject arbitrary code into your publish pipeline.

### Trigger mechanism

The workflow fires on `v*` tag pushes. This pairs naturally with a release skill (per [P017](patterns/P017-release-skill.md)) that bumps the version, writes the changelog, publishes (or in this case, pushes the tag which triggers this workflow), and verifies CI green. The tag push is the release signal — the workflow does the rest.

## Anti-patterns to avoid

- **Using `NPM_TOKEN` alongside OIDC.** If a `NODE_AUTH_TOKEN` env var is set (from a repository secret), the npm CLI uses it instead of OIDC. Remove the secret and the env var — OIDC takes over.
- **Forgetting `id-token: write`.** Without this permission, the OIDC token isn't generated and `npm publish` fails with a cryptic auth error. It must be a top-level `permissions` block, not `secrets.GITHUB_TOKEN`.
- **Wrong `registry-url`.** If `registry-url` is missing from the Node setup step, `npm publish` defaults to `https://registry.npmjs.org` anyway — but being explicit avoids confusion if someone later adds a private registry.
- **Using `--provenance` flag unnecessarily.** For public packages in public repos, provenance is generated automatically by OIDC trusted publishing. The flag is only needed when publishing from private repos or when you want to enforce provenance explicitly. Omit it when OIDC is doing the work.
- **Trusted publisher environment mismatch.** If you configure a GitHub environment (e.g. `release`) on npmjs.com, the workflow must use `environment: release` in the job. If you leave the environment blank on npmjs.com, the workflow must not set one. They must match exactly.
