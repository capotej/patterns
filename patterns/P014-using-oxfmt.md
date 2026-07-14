---
id: P014
name: using-oxfmt
date: 2026-07-14
revision: 1
author: Julio Capote
used_in: []
---

# Using oxfmt

oxfmt is the JS/TS/JSON formatter from the oxc project. It replaces Prettier (or the format half of Biome, P005) with a single fast Rust binary. It's configured via a committed `.oxfmtrc.jsonc` and exposed as `pnpm format` (write) / `pnpm format:check` (read-only CI gate).

There are **two equally valid install paths**, and the choice is per-project. The config file and scripts are identical either way — only the provisioning differs:

- **devDependency (default).** `oxfmt` is a normal npm package, installed via `pnpm add -D oxfmt`, resolved from `node_modules/.bin`, version-pinned by the lockfile. This is the path the official docs lead with and the natural choice for any JS/TS project — a 1:1 drop-in for how Biome (P005) is installed.
- **mise tool (alternative).** Declared in `mise.toml` (P003) and fetched from the `oxc-project/oxc` GitHub release. Use this when the project is already mise-based and wants the formatter *outside* `node_modules` (it survives `rm -rf node_modules`, runs before `pnpm install`, and shares a release tag with oxlint). Covered at the end of this pattern.

## When to use

- Any JS/TS project that wants a fast, single-binary formatter.
- Any project already following P004 (pnpm) — install `oxfmt` as a devDependency, exactly like Biome (P005).
- Any project already following P003 (mise) that prefers its non-runtime tools in `mise.toml` rather than `node_modules` (the alternative path below).
- Any project pairing it with oxlint (P015) — the two are the oxc toolchain's format and lint halves.

## How it works

### Install path A: devDependency (default)

Install as a devDependency and pin it to an exact version (no caret). Note `oxfmt` is its own npm package, versioned independently of `oxlint`:

```json title="package.json"
{
  "devDependencies": {
    "oxfmt": "0.59.0"
  }
}
```

```json title="package.json"
// Wrong — a caret lets the installed version drift between machines and CI
{
  "devDependencies": {
    "oxfmt": "^0.59.0"
  }
}
```

After `pnpm install`, `oxfmt` resolves from `node_modules/.bin` and the pnpm scripts below invoke it by bare name. This is the same model as Biome (P005) — no mise, no asset selection, no separate install step in CI (it lands in `node_modules` during `pnpm install --frozen-lockfile`). Prefer this path unless you have a specific reason to keep the formatter out of `node_modules`.

### Install path B: mise tool (alternative)

If the project is already on mise (P003) and wants the formatter outside `node_modules`, declare it in `mise.toml` instead. `oxfmt` is then fetched from the `oxc-project/oxc` GitHub release — there is no `oxfmt` entry under `devDependencies`, and the formatter survives an `rm -rf node_modules`.

**The shared-release mechanism.** oxfmt and oxlint are published together inside a single `oxc-project/oxc` GitHub release tagged `apps_v<N>` (e.g. `apps_v1.73.0` bundles oxlint 1.73.0 **and** oxfmt 0.58.0). This is the one thing the mise path gives you that the npm path does not: both tools come from one release, so they can never drift apart. On npm the two packages are independent and can move separately; on mise they're pinned to one shared tag.

```toml title="mise.toml"
# oxfmt (formatter) — ships alongside oxlint in the oxc-project/oxc release
# (tag apps_v<N> bundles matching oxlint + oxfmt versions). A distinct
# [tool_alias] per binary is REQUIRED: matching/rename_exe are not part of
# the install key, so two entries pointing at the same github backend without
# distinct aliases resolve to the same dir and the second install overwrites
# the first.
[tool_alias]
oxlint = "github:oxc-project/oxc"
oxfmt = "github:oxc-project/oxc"

[tools.oxfmt]
version = "apps_v1.73.0"   # MUST equal [tools.oxlint].version
rename_exe = "oxfmt"

[tools.oxfmt.platforms]
linux-x64   = { asset_pattern = "oxfmt-x86_64-unknown-linux-gnu.tar.gz" }
linux-arm64 = { asset_pattern = "oxfmt-aarch64-unknown-linux-gnu.tar.gz" }
macos-x64   = { asset_pattern = "oxfmt-x86_64-apple-darwin.tar.gz" }
macos-arm64 = { asset_pattern = "oxfmt-aarch64-apple-darwin.tar.gz" }
```

Three details are load-bearing on this path:

- **Distinct `[tool_alias]` per binary** — both point at `github:oxc-project/oxc`, so without distinct aliases mise installs them into the same directory and one overwrites the other.
- **`rename_exe = "oxfmt"`** — strips the target-triple suffix so the binary is invokable as plain `oxfmt`.
- **Per-platform `asset_pattern`** (under `[tools.oxfmt.platforms]`) — pins the literal archive name per platform. Prefer this over `matching`/`matching_regex`, which mis-pick the asset in some mise revisions (silently resolving both binaries to oxfmt). Every target platform must be listed explicitly; `asset_pattern` replaces autodetection, so an unlisted platform has no fallback.

Bump both aliases together: change the single `version = "apps_v<N>"` line for **both** `oxfmt` and `oxlint` to the same tag, then `mise install`. See P015 for the matching `oxlint` alias. The full CI provisioning for this path uses `jdx/mise-action` (P016).

### Commit .oxfmtrc.jsonc — JSONC, not JSON (either path)

The config lives at the repo root as `.oxfmtrc.jsonc`, regardless of install path. Use `.jsonc` deliberately: the `//` comments let you annotate every non-default choice (what's ignored, what's turned off) with its rationale.

```jsonc title=".oxfmtrc.jsonc"
{
  // oxfmt config — the JS/TS/JSON formatter. Run via `pnpm format` (--write)
  // or `pnpm format:check` (--check, CI gate). The style values below are
  // oxfmt's defaults, spelled out explicitly as the formatter contract:
  // printWidth 100 / 2-space indent / semicolons / double quotes.
  "$schema": "https://raw.githubusercontent.com/oxc-project/oxc/main/npm/oxfmt/configuration_schema.json",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": false,

  // OFF: oxfmt sorts package.json keys by default, which would alphabetize
  // package.json's deliberate hand-curated field order (name/version/type at
  // the top; keywords/license/repository at the bottom). Format the JSON,
  // but do NOT reorder its keys.
  "experimentalSortPackageJson": false,

  // What oxfmt does NOT touch:
  //   - dist, generated/    compiler output + any large bundled/vendored tree
  //   - .oxfmtrc.jsonc,     hand-curated tool config (this file; oxlint
  //     .oxlintrc.json,     config; markdownlint config). oxfmt's default
  //     .markdownlint-      trailingComma "all" would add trailing commas to
  //     cli2.jsonc          these .jsonc files — a JSONC trailing comma risks
  //                          breaking markdownlint-cli2's parser and is
  //                          circular for oxfmt to reformat its own config.
  //                          (Strict .json manifests like package.json /
  //                          tsconfig.json stay in scope — JSON forbids
  //                          trailing commas, so they pass clean.)
  "ignorePatterns": [
    "dist",
    "generated",
    ".oxfmtrc.jsonc",
    ".oxlintrc.json",
    ".markdownlint-cli2.jsonc"
  ]
}
```

Three things this config is really doing:

1. **Documenting the style contract.** `printWidth`/`tabWidth`/`useTabs`/`semi`/`singleQuote` happen to be oxfmt's defaults; they're stated explicitly so the style is a documented decision, not an accident of defaults. State your project's actual style — these are per-project, the way Biome's `indentStyle` is (P005).
2. **Preserving `package.json` field order.** `experimentalSortPackageJson: false` is the load-bearing one. Without it, oxfmt reorders `package.json` keys alphabetically and destroys a hand-curated field layout. Leave it off whenever the project cares about its `package.json` key order.
3. **Keeping oxfmt off the hand-curated `.jsonc` configs.** oxfmt's default `trailingComma: "all"` adds trailing commas, which are valid JSONC but can break other parsers (markdownlint-cli2) and make oxfmt reformat its own config circularly. List those dotfiles in `ignorePatterns`. Strict `.json` files (`package.json`, `tsconfig.json`) stay in scope because JSON forbids trailing commas and they pass clean.

### The format / format:check scripts (either path)

Exposed as pnpm scripts (invoked via pnpm per P004). Identical regardless of how `oxfmt` was installed — both paths put `oxfmt` on PATH:

```json title="package.json"
{
  "scripts": {
    "format": "oxfmt --write .",
    "format:check": "oxfmt --check ."
  }
}
```

- `format` (`oxfmt --write .`) — applies formatting in place. The developer/agent fixer.
- `format:check` (`oxfmt --check .`) — read-only; exits non-zero on unformatted files. **This is the CI gate.**

The split is deliberate: CI never writes, it only checks. Agents and developers run `pnpm format` to fix; CI runs `pnpm format:check` to verify.

### Gate format:check in CI

Where CI exists, `pnpm format:check` is a required step, separate from the build and from lint:

```yaml title=".github/workflows/ci.yml"
- run: pnpm install --frozen-lockfile
- run: pnpm format:check
```

How `oxfmt` gets there depends on the install path: on the devDependency path it lands in `node_modules` during `pnpm install` (no extra step); on the mise path it's provisioned from `mise.toml` via `jdx/mise-action` (P016). The workflow actions are SHA-pinned per P002.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Formatting

This project formats JS/TS/JSON with **oxfmt**. Do not introduce prettier, biome, or editor-specific config.

- Format in place: `pnpm format` (runs `oxfmt --write .`)
- Check only (CI gate): `pnpm format:check` (runs `oxfmt --check .`)

Configuration lives in `.oxfmtrc.jsonc`. `oxfmt` is installed as a pnpm devDependency (or, in mise-based projects, declared in `mise.toml` and installed via `mise install`); in either case invoke it by bare name through pnpm. The config disables `experimentalSortPackageJson` to preserve `package.json` field order, and ignores the hand-curated `.jsonc` config files.
```

## Anti-patterns to avoid

- **Pinning with a caret.** `"oxfmt": "^0.59.0"` drifts between machines and CI. Pin exactly, like Biome (P005).
- **Mixing install paths.** Pick one home for `oxfmt`: either the devDependency or `mise.toml`, not both. Declaring it in both means two versions that can drift, and defeats the point of either choice.
- **Putting oxlint and oxfmt on different `apps_v<N>` tags (mise path only).** They ship in one release; the point of the shared tag is that formatter and linter move together. Always set both `[tools.oxfmt].version` and `[tools.oxlint].version` to the same tag.
- **Omitting `rename_exe` (mise path only).** Without it the binary lands as `oxfmt-x86_64-unknown-linux-gnu`, so `oxfmt` is not on PATH and every script fails.
- **Relying on `matching`/`matching_regex` instead of per-platform `asset_pattern` (mise path only).** Those selectors mis-pick the asset in some mise revisions. List each target platform explicitly under `[tools.oxfmt.platforms]`.
- **Leaving `experimentalSortPackageJson` at its default.** oxfmt will alphabetize `package.json` keys and clobber a deliberate field order. Turn it off whenever the project hand-curates its `package.json` layout.
- **Letting oxfmt reformat the `.jsonc` config files.** The default `trailingComma: "all"` adds trailing commas that can break other JSONC parsers and make oxfmt edit its own config. Add `.oxfmtrc.jsonc`, `.oxlintrc.json`, and any other hand-authored `.jsonc` to `ignorePatterns`.
- **Formatting generated output.** `dist/` and any large bundled or vendored tree must be in `ignorePatterns`. Format failures in generated code are noise that trains people to ignore the formatter.
- **A `format` script that writes, used as the CI gate.** CI must use `format:check` (`--check`), the read-only gate. A writing step in CI is both pointless and dangerous.
- **Introducing prettier/biome alongside oxfmt.** oxfmt owns formatting. A second formatter produces conflicting output and duplicated config. (oxlint, P015, owns linting — same project, different job.)
