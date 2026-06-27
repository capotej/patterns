---
id: P004
name: using-pnpm
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/harness', 'github.com/capotej/pi-mise']
---

# Using pnpm

pnpm is the package manager, declared once via the `packageManager` field in `package.json` and pinned to an exact version. The lockfile is committed for reproducible installs; the local store cache is gitignored.

## When to use

- Any Node/TypeScript project that needs fast, disk-efficient, reproducible dependency installs.
- Any project where developers, CI, and containerized agents must resolve the exact same package-manager version and dependency tree.
- Any project used inside containerized agent environments (harness, pi-mise) where pnpm's content-addressable store should be cached inside the working directory.

## How it works

Three things live in the repo root and together make pnpm the source of truth: a `packageManager` field, a committed `pnpm-lock.yaml`, and a `.gitignore` entry for `.pnpm-store`.

### Declare the package-manager version in `package.json`

The `packageManager` field is the single declaration of which pnpm version the project uses. Node's corepack (bundled since Node 16.10) reads this field, so any corepack-enabled environment automatically runs exactly that pnpm version — no separate install step, no `.nvmrc`-style drift:

```json title="package.json"
{
  "name": "@boldblackai/harness",
  "packageManager": "pnpm@10.33.2"
}
```

```json title="package.json"
// Wrong — version ranges or "latest" defeat the purpose of pinning
{
  "packageManager": "pnpm@^10"
}

// Wrong — leaving it out means every environment resolves its own pnpm
{
  "packageManager": "pnpm"
}
```

**Which version to pin:** take the latest stable pnpm release and pin that exact version. The specific number is not important — what matters is that one exact version is declared. Find the latest with `npm view pnpm version`, then set `"packageManager": "pnpm@<version>"`. (That both reference repos use `10.33.2` is a coincidence, not a coordination rule.)

### Commit the lockfile

`pnpm-lock.yaml` is committed to the repo. It records the exact resolved dependency tree. Every install — local, CI, agent — reproduces the same versions:

```bash
pnpm install --frozen-lockfile   # CI / reproducible installs
pnpm install                      # dev — updates the lockfile only if package.json changed
```

### Gitignore the local store

pnpm stores every package once in a content-addressable store and hardlinks from there into `node_modules`. When the store is configured to live inside the project (common in CI and containers so the cache is co-located with the checkout), it lands in `.pnpm-store/`. That directory is a machine-generated cache and must never be committed:

```gitignore title=".gitignore"
node_modules/
.pnpm-store
```

```gitignore title=".gitignore"
// Wrong — committing the cache bloats the repo and is machine-specific
// .pnpm-store  (must be ignored, not committed)
```

Note the asymmetry: `pnpm-lock.yaml` is committed (it's the reproducibility contract); `.pnpm-store` is ignored (it's a derived cache).

### CI: action-setup + setup-node, then frozen-lockfile install

CI does not rely on a pre-installed pnpm. It installs the exact version and wires pnpm into the node dependency cache:

```yaml title=".github/workflows/e2e.yml"
steps:
  - uses: actions/checkout@93cb6efe18208431cddfb8368fd83d5badbf9bfd # v5

  - name: Setup pnpm
    uses: pnpm/action-setup@0e279bb959325dab635dd2c09392533439d90093 # v6

  - name: Setup Node ${{ matrix.node }}
    uses: actions/setup-node@48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e # v6
    with:
      node-version: ${{ matrix.node }}
      cache: pnpm

  - name: Install
    run: pnpm install --frozen-lockfile
```

Key points:

1. **`pnpm/action-setup` with no explicit `version`** reads the version from the `packageManager` field. Pin the action by SHA (see P002). There is exactly one declaration of the pnpm version and CI follows it.
2. **`actions/setup-node` with `cache: pnpm`** enables dependency caching keyed on `pnpm-lock.yaml`. The `setup-pnpm` step must come before `setup-node` so the caching can find pnpm.
3. **`pnpm install --frozen-lockfile`** refuses to mutate the lockfile — CI fails fast if `package.json` and `pnpm-lock.yaml` disagree.

### Scripts call pnpm explicitly

Project scripts invoke pnpm (and pnpm-managed binaries via `pnpm exec`) so the correct tool resolution is used everywhere:

```json title="package.json"
{
  "scripts": {
    "lint": "pnpm lint:ts && pnpm lint:md && pnpm lint:sh",
    "lint:ts": "biome check .",
    "build": "tsc"
  }
}
```

## The AGENTS.md section

Add this to your agent instructions so coding agents use pnpm correctly:

```markdown title="AGENTS.md"
## Package Manager

This project uses **pnpm**, declared via the `packageManager` field in `package.json`. Do not use npm or yarn.

- Install dependencies: `pnpm install`
- Add a dependency: `pnpm add <pkg>` (this updates `pnpm-lock.yaml`)
- Add a dev dependency: `pnpm add -D <pkg>`
- Run a script: `pnpm <script>` or `pnpm run <script>`
- Run a local binary: `pnpm exec <bin>` (e.g. `pnpm exec tsc --noEmit`)
- In CI or to enforce the lockfile: `pnpm install --frozen-lockfile`

The pnpm version is pinned in `package.json` (`"packageManager": "pnpm@x.y.z"`) and read automatically by corepack. Do not change it unless asked. `pnpm-lock.yaml` is committed and is the source of truth for the dependency tree; `.pnpm-store/` is a cache and is gitignored.
```

## Anti-patterns to avoid

- **Committing `.pnpm-store/`.** It is a content-addressable cache, machine-generated and potentially large. It belongs in `.gitignore`, next to `node_modules/`. Only `pnpm-lock.yaml` is committed.
- **Unpinning the `packageManager` version.** `"packageManager": "pnpm@^10"` or omitting the version lets each environment resolve a different pnpm, producing different lockfiles and install behavior. Pin to an exact version: `pnpm@10.33.2`.
- **Using `npm install` or `yarn` in scripts or docs.** Mixing package managers corrupts the lockfile and `node_modules` layout. Every install/add/run goes through `pnpm`.
- **Running `pnpm install` (without `--frozen-lockfile`) in CI.** CI should fail if the lockfile is out of sync with `package.json`, not silently rewrite it. Use `--frozen-lockfile`.
- **Declaring pnpm as a mise tool.** pnpm is managed by corepack (which ships with Node) via the `packageManager` field, not by mise. See P003: package managers are not separate mise tools. Declaring `pnpm` in `mise.toml [tools]` creates two competing sources of truth.
- **Putting `setup-node` before `pnpm/action-setup` in CI.** `setup-node`'s `cache: pnpm` needs pnpm on PATH to locate the cache, so the order matters.

## Reference: pnpm configuration across repos

### boldblackai/harness

```json title="package.json"
{
  "name": "@boldblackai/harness",
  "version": "1.9.0",
  "bin": { "harness": "bin/harness.js" },
  "packageManager": "pnpm@10.33.2",
  "scripts": {
    "lint": "pnpm lint:ts && pnpm lint:md && pnpm lint:sh && pnpm lint:docker && pnpm lint:actions",
    "lint:ts": "biome check .",
    "test:e2e": "node --test tests/e2e/*.test.mjs",
    "build": "tsc && cp -r src/seccomp-profiles bin/seccomp-profiles"
  },
  "devDependencies": {
    "@biomejs/biome": "1.9.4",
    "markdownlint-cli2": "0.17.2",
    "typescript": "^6.0.2"
  }
}
```

```yaml title=".github/workflows/e2e.yml"
- name: Setup pnpm
  uses: pnpm/action-setup@0e279bb959325dab635dd2c09392533439d90093 # v6
- name: Setup Node ${{ matrix.node }}
  uses: actions/setup-node@48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e # v6
  with:
    node-version: ${{ matrix.node }}
    cache: pnpm
- name: Install
  run: pnpm install --frozen-lockfile
```

A containerized agent runner. Pins pnpm to `10.33.2`, commits `pnpm-lock.yaml`, and uses `pnpm/action-setup` + `actions/setup-node` (`cache: pnpm`) in both its `e2e.yml` and `lint.yml` workflows. The action is pinned by SHA per P002. Node itself is provisioned by `actions/setup-node` in CI rather than by mise.

### capotej/pi-mise

```json title="package.json"
{
  "name": "@capotej/pi-mise",
  "packageManager": "pnpm@10.33.2",
  "scripts": {
    "build": "tsc",
    "lint": "biome check .",
    "format": "biome format --write ."
  }
}
```

```gitignore title=".gitignore"
node_modules/
.pnpm-store
dist
```

A pi extension. Has no CI workflows; the convention is enforced by the `packageManager` field plus corepack locally and the committed lockfile.
