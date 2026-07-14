---
id: P014
name: using-oxfmt
date: 2026-07-14
author: Julio Capote
used_in: []
---

# Using oxfmt

oxfmt is the JS/TS/JSON formatter from the oxc project. Unlike Biome (P005), which is an npm devDependency, **oxfmt is a mise tool**: it is declared in `mise.toml` (P003) and installed from the `oxc-project/oxc` GitHub release, not pulled from `node_modules`. It is configured via a committed `.oxfmtrc.jsonc` and exposed as `pnpm format` (write) / `pnpm format:check` (read-only CI gate).

## When to use

- Any JS/TS project that wants a fast, single-binary formatter without adding an npm devDependency for it.
- Any project already following P003 (mise) — oxfmt is just another entry in `mise.toml`, installed by the same mechanism as the rest of the toolchain.
- Any project that wants the formatter and linter to come from the same release (oxfmt and oxlint, P015, ship together — see below) so they can never drift apart.
- An alternative to Biome (P005) when you'd rather the formatter not live in `node_modules`.

## How it works

### oxfmt is a mise tool, not an npm devDependency

The formatter does not go in `package.json` `devDependencies`. It is a GitHub-release binary fetched by mise, exactly like `shellcheck` or `actionlint` in P003. This is the single biggest difference from the Biome convention (P005): there is no `oxfmt` entry under `devDependencies`, no `oxfmt` in `node_modules/.bin`, and the formatter survives an `rm -rf node_modules`. The downside is it must be installed via `mise install` (or provisioned in CI — see P016), not `pnpm install`.

### One release ships both oxfmt and oxlint

oxfmt and oxlint are published together inside a single `oxc-project/oxc` GitHub release. The release tag is `apps_v<N>` (e.g. `apps_v1.73.0` bundles oxlint 1.73.0 **and** oxfmt 0.58.0). They are not independently versioned and never move apart: pinning both to the same `apps_v<N>` tag guarantees the formatter and linter came from the same build. This is why both aliases below share one `version`.

### Declare both binaries in mise.toml with distinct `[tool_alias]` entries

oxfmt and oxlint resolve to the **same** GitHub backend (`github:oxc-project/oxc`). Because the backend is identical, mise needs a **distinct `[tool_alias]` per binary** so each installs into its own directory:

```toml title="mise.toml"
# Oxlint (linter) + oxfmt (formatter) — replace Biome (P005). Both ship as
# per-platform archives inside a single oxc-project/oxc release whose tag is
# `apps_v<N>` (apps_v1.73.0 bundles oxlint 1.73.0 and oxfmt 0.58.0). A
# distinct [tool_alias] per binary is REQUIRED: matching/rename_exe are not
# part of the install key, so two entries pointing at the same github backend
# without distinct aliases resolve to the same dir and the second install
# overwrites the first.
[tool_alias]
oxlint = "github:oxc-project/oxc"
oxfmt = "github:oxc-project/oxc"

[tools.oxfmt]
version = "apps_v1.73.0"
rename_exe = "oxfmt"

[tools.oxfmt.platforms]
linux-x64   = { asset_pattern = "oxfmt-x86_64-unknown-linux-gnu.tar.gz" }
linux-arm64 = { asset_pattern = "oxfmt-aarch64-unknown-linux-gnu.tar.gz" }
macos-x64   = { asset_pattern = "oxfmt-x86_64-apple-darwin.tar.gz" }
macos-arm64 = { asset_pattern = "oxfmt-aarch64-apple-darwin.tar.gz" }
```

Two details matter:

- **`rename_exe = "oxfmt"`** — the archive's binary is named with a target-triple suffix (`oxfmt-x86_64-unknown-linux-gnu`). `rename_exe` strips the suffix so the binary is invokable as plain `oxfmt`.
- **Per-platform `asset_pattern`** (under `[tools.oxfmt.platforms]`) selects the exact archive by a literal per-platform filename. Prefer `asset_pattern` over `matching`/`matching_regex`: the latter select the wrong asset in some mise revisions (silently resolving both binaries to oxfmt), whereas `asset_pattern` pins the literal archive name. Every platform you intend to support must be listed explicitly — `asset_pattern` replaces autodetection entirely, so an unlisted platform has no fallback.

### Bump both aliases together

Because one `apps_v<N>` tag carries both binaries, a version bump is a single line changed in two places — the shared version, on both the `oxlint` and `oxfmt` aliases:

```toml title="mise.toml"
[tools.oxfmt]
version = "apps_v1.74.0"   # bump this AND [tools.oxlint].version together
```

Change both `version = "apps_v<N>"` lines to the same tag, then `mise install`. Never put the two aliases on different `apps_v` tags — that defeats the "formatter and linter ship together" guarantee.

### Commit .oxfmtrc.jsonc — JSONC, not JSON

The config lives at the repo root as `.oxfmtrc.jsonc`. Use `.jsonc` deliberately: the `//` comments let you annotate every non-default choice (what's ignored, what's turned off) with its rationale. A disable or ignore with no comment is a gap.

```jsonc title=".oxfmtrc.jsonc"
{
  // oxfmt config — the JS/TS/JSON formatter. See mise.toml for the tool
  // install (oxfmt ships in the oxc-project/oxc github release, fetched by
  // mise; not an npm devDependency). Run via `pnpm format` (--write) or
  // `pnpm format:check` (--check, CI gate).
  //
  // The style values below are oxfmt's defaults, spelled out explicitly as
  // the formatter contract: printWidth 100 / 2-space indent / semicolons /
  // double quotes. State them so the style is documented, not inferred.
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

### Three things the config is really doing

1. **Documenting the style contract.** `printWidth`/`tabWidth`/`useTabs`/`semi`/`singleQuote` happen to be oxfmt's defaults; they're stated explicitly so the style is a documented decision, not an accident of defaults. State your project's actual style — these are per-project, the way Biome's `indentStyle` is (P005).
2. **Preserving `package.json` field order.** `experimentalSortPackageJson: false` is the load-bearing one. Without it, oxfmt reorders `package.json` keys alphabetically and destroys a hand-curated field layout. Leave it off whenever the project cares about its `package.json` key order.
3. **Keeping oxfmt off the hand-curated `.jsonc` configs.** oxfmt's default `trailingComma: "all"` adds trailing commas, which are valid JSONC but can break other parsers (markdownlint-cli2) and make oxfmt reformat its own config circularly. List those dotfiles in `ignorePatterns`. Strict `.json` files (`package.json`, `tsconfig.json`) stay in scope because JSON forbids trailing commas and they pass clean.

### The format / format:check scripts

Exposed as pnpm scripts (invoked via pnpm per P004):

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

oxfmt itself is provisioned in CI from the same `mise.toml` via `jdx/mise-action` (P016), so the formatter version in CI is identical to local — there is no second install step to drift. The workflow actions are SHA-pinned per P002.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Formatting

This project formats JS/TS/JSON with **oxfmt**. Do not introduce prettier, biome, or editor-specific config.

- Format in place: `pnpm format` (runs `oxfmt --write .`)
- Check only (CI gate): `pnpm format:check` (runs `oxfmt --check .`)

Configuration lives in `.oxfmtrc.jsonc`. oxfmt is a **mise tool** (declared in `mise.toml`), not an npm devDependency — it installs with `mise install`, not `pnpm install`. It ships in the same `oxc-project/oxc` release as oxlint, so both share one `apps_v<N>` version; bump both aliases together and run `mise install`. The config disables `experimentalSortPackageJson` to preserve `package.json` field order, and ignores the hand-curated `.jsonc` config files.
```

## Anti-patterns to avoid

- **Adding `oxfmt` as an npm devDependency.** oxfmt is a mise tool (P003). A `node_modules/.bin/oxfmt` is a different install path than the one `mise.toml` declares, so the two can diverge. Keep it in `mise.toml` only.
- **Splitting oxfmt and oxlint across different `apps_v<N>` tags.** They ship in one release; the whole point of the shared tag is that formatter and linter move together. Always set both `[tools.oxfmt].version` and `[tools.oxlint].version` to the same tag.
- **Omitting `rename_exe`.** Without it the binary lands as `oxfmt-x86_64-unknown-linux-gnu` (or similar), so `oxfmt` is not on PATH and every script fails. `rename_exe = "oxfmt"` strips the target-triple suffix.
- **Relying on `matching`/`matching_regex` instead of per-platform `asset_pattern`.** Those selectors mis-pick the asset in some mise revisions (resolving both oxfmt and oxlint to oxfmt). List each target platform explicitly under `[tools.oxfmt.platforms]` with a literal `asset_pattern`.
- **Leaving `experimentalSortPackageJson` at its default.** oxfmt will alphabetize `package.json` keys and clobber a deliberate field order. Turn it off whenever the project hand-curates its `package.json` layout.
- **Letting oxfmt reformat the `.jsonc` config files.** The default `trailingComma: "all"` adds trailing commas that can break other JSONC parsers and make oxfmt edit its own config. Add `.oxfmtrc.jsonc`, `.oxlintrc.json`, and any other hand-authored `.jsonc` to `ignorePatterns`.
- **Linting/formatting generated output.** `dist/` and any large bundled or vendored tree must be in `ignorePatterns`. Format failures in generated code are noise that trains people to ignore the formatter.
- **A `format` script that writes, used as the CI gate.** CI must use `format:check` (`--check`), the read-only gate. A writing step in CI is both pointless and dangerous.
- **Introducing prettier/biome alongside oxfmt.** oxfmt owns formatting. A second formatter produces conflicting output and duplicated config. (oxlint, P015, owns linting — the two are complements from the same release, not alternatives.)
