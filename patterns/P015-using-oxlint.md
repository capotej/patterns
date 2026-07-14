---
id: P015
name: using-oxlint
date: 2026-07-14
revision: 1
author: Julio Capote
used_in: []
---

# Using oxlint

oxlint is the JS/TS linter from the oxc project. It replaces ESLint (or the lint half of Biome, P005) with a single fast Rust binary. It's configured via a committed `.oxlintrc.json` and exposed as `pnpm lint` (CI gate) / `pnpm lint:fix`.

Like oxfmt (P014), oxlint has **two equally valid install paths** — a devDependency (the default, what the official docs lead with) or a mise tool (P003). The config file and scripts are identical either way; only the provisioning differs. See P014 for the full install-path comparison — this pattern focuses on the lint config and assumes you've chosen a path there.

## When to use

- Any JS/TS project that wants a fast linter.
- Any project already following P004 (pnpm) — install `oxlint` as a devDependency, exactly like Biome (P005).
- Any project already following P003 (mise) that prefers its non-runtime tools in `mise.toml` rather than `node_modules` (the alternative path).
- Any project pairing it with oxfmt (P014) — the two are the oxc toolchain's lint and format halves.

## How it works

### Install — pick a path (see P014)

oxlint is installed the same way oxfmt is; the rationale for choosing between the two paths is in P014. In short:

- **devDependency (default):** `pnpm add -D oxlint`, pinned exactly. `oxlint` resolves from `node_modules/.bin`. Same model as Biome (P005).

  ```json title="package.json"
  {
    "devDependencies": {
      "oxlint": "1.74.0"
    }
  }
  ```

  Note `oxlint` and `oxfmt` are **independent npm packages** with independent versions — pin each on its own line; they do not share a version on npm.

- **mise tool (alternative):** declared in `mise.toml` via a second `[tool_alias]` pointing at `github:oxc-project/oxc`, sharing the same `apps_v<N>` tag as oxfmt (so the two bump together). The matching alias block:

  ```toml title="mise.toml"
  [tool_alias]
  oxlint = "github:oxc-project/oxc"
  oxfmt  = "github:oxc-project/oxc"

  [tools.oxlint]
  version = "apps_v1.73.0"   # MUST equal [tools.oxfmt].version
  rename_exe = "oxlint"

  [tools.oxlint.platforms]
  linux-x64   = { asset_pattern = "oxlint-x86_64-unknown-linux-gnu.tar.gz" }
  linux-arm64 = { asset_pattern = "oxlint-aarch64-unknown-linux-gnu.tar.gz" }
  macos-x64   = { asset_pattern = "oxlint-x86_64-apple-darwin.tar.gz" }
  macos-arm64 = { asset_pattern = "oxlint-aarch64-apple-darwin.tar.gz" }
  ```

  The full install-path rules (distinct aliases, `rename_exe`, per-platform `asset_pattern`, bump-both-together) are in P014; CI provisioning for this path uses `jdx/mise-action` (P016).

### Commit .oxlintrc.json — JSONC, not JSON (either path)

The config lives at the repo root, regardless of install path. Write it as JSONC (oxlint parses `//` comments) so every category choice and rule override carries its rationale inline.

```jsonc title=".oxlintrc.json"
{
  // oxlint config — the JS/TS linter. Run via `pnpm lint` (CI gate),
  // `pnpm lint:fix`, or the lint suite script.
  //
  // Categories: correctness, suspicious, and perf are all fatal hard gates.
  //   correctness = definitely wrong; suspicious = likely wrong;
  //   perf = runtime performance. This is the strict posture: anything
  //   oxlint can prove is wrong or wasteful blocks CI. Code that is
  //   intentionally sequential or holds a deliberate pattern suppresses the
  //   specific rule inline with a named reason (see the no-underscore-dangle
  //   override below), keeping the rule active globally.
  "$schema": "https://raw.githubusercontent.com/oxc-project/oxc/main/npm/oxlint/configuration_schema.json",
  "categories": {
    "correctness": "error",
    "suspicious": "error",
    "perf": "error"
  },
  "env": {
    "node": true
  },
  "rules": {
    // __dirname/__filename are Node ESM-via-fileURLToPath conventions, not
    // author-chosen dangling-underscore identifiers. Allow them so the rule
    // (still active for all other _-prefixed names) doesn't flag the idiom.
    "eslint/no-underscore-dangle": ["error", { "allow": ["__dirname", "__filename"] }]
  },
  // Generated output and any bundled/vendored snapshot are never linted.
  // (node_modules/, .pnpm-store/, .jj/, .git/ are already gitignored, which
  // oxlint respects; dist/ listed for clarity alongside generated/.)
  "ignorePatterns": ["dist", "generated"]
}
```

### The three category levels, and how to adopt them

oxlint groups its rules into categories, each set to `"error"`, `"warn"`, or `"off"`. The strict posture is all three at `"error"`:

```jsonc title=".oxlintrc.json"
{
  "categories": {
    "correctness": "error",
    "suspicious": "error",
    "perf": "error"
  }
}
```

- **`correctness`** — the code is definitely wrong. This is parity with Biome's `recommended` set (P005): Biome only covered correctness-class checks. Start here; it should pass clean on any reasonable codebase.
- **`suspicious`** — the code is likely wrong (sequential `await` in a loop, `.sort()` mutating in place, swallowing caught errors). Real findings; worth gating.
- **`perf`** — runtime-performance issues. Wasteful but not incorrect.

Because turning suspicious/perf straight to `"error"` can surface a backlog on an existing codebase, the recommended adoption order is **staged**:

```jsonc title=".oxlintrc.json"
// Phase 1 — correctness only is a hard gate (matches Biome's coverage):
{ "categories": { "correctness": "error", "suspicious": "warn", "perf": "warn" } }

// Phase 2 — once the warnings are resolved, promote both to error:
{ "categories": { "correctness": "error", "suspicious": "error", "perf": "error" } }
```

Run with `suspicious`/`perf` at `"warn"` first, resolve every finding (oxlint reports them but does not fail CI), then flip both to `"error"` in a dedicated change. Do not leave them at `"warn"` permanently — a warning that never blocks is a warning that gets ignored.

### Set the env

```jsonc title=".oxlintrc.json"
{
  "env": { "node": true }
}
```

`env.node: true` tells oxlint which globals are defined, so Node built-ins (`process`, `__dirname`, etc.) are not flagged as undefined. Set the env that matches your runtime (node, browser, etc.).

### Suppress a rule surgically, with a reason

The default is to keep every category active globally and override **individual** rules only where the project has a legitimate exception — and to name the reason inline. A real example: `__dirname`/`__filename` are Node ESM conventions (reconstructed via `fileURLToPath`), not author-chosen dangling-underscore names, so the rule is allowed to ignore exactly those two while still catching every other `_`-prefixed identifier:

```jsonc title=".oxlintrc.json"
{
  "rules": {
    "eslint/no-underscore-dangle": ["error", { "allow": ["__dirname", "__filename"] }]
  }
}
```

Prefer this over turning a category off. The rule stays on for the whole codebase; only the named, justified exception is carved out. Inline-disable comments in source work the same way — they suppress one occurrence and should carry a reason, the same discipline as markdownlint disables (P007).

### The lint / lint:fix scripts (either path)

Exposed as pnpm scripts (invoked via pnpm per P004). Identical regardless of how `oxlint` was installed:

```json title="package.json"
{
  "scripts": {
    "lint": "oxlint .",
    "lint:fix": "oxlint --fix ."
  }
}
```

- `lint` (`oxlint .`) — runs the linter; exits non-zero on `"error"`-level findings. **This is the CI gate.**
- `lint:fix` (`oxlint --fix .`) — applies safe auto-fixes in place.

`lint` is often composed with the other lint arms into a single entry point so the whole pass is one command:

```json title="package.json"
{
  "scripts": {
    "lint": "oxlint . && pnpm lint:md",
    "lint:md": "markdownlint-cli2 \"**/*.md\"",
    "lint:fix": "oxlint --fix . && markdownlint-cli2 --fix \"**/*.md\""
  }
}
```

Here `lint` runs oxlint (JS/TS) then markdownlint-cli2 (Markdown, P007). oxlint owns JS/TS the way Biome did (P005); markdownlint owns Markdown. One `pnpm lint` fans out to every file type.

### Gate lint in CI

Where CI exists, `pnpm lint` is a required step:

```yaml title=".github/workflows/ci.yml"
- run: pnpm install --frozen-lockfile
- run: pnpm lint
```

How `oxlint` gets there depends on the install path: on the devDependency path it lands in `node_modules` during `pnpm install`; on the mise path it's provisioned from `mise.toml` via `jdx/mise-action` (P016). Workflow actions are SHA-pinned per P002.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Linting

This project lints JS/TS with **oxlint**. Do not introduce eslint, biome, or editor-specific config.

- Lint (CI gate): `pnpm lint` (runs `oxlint .`)
- Auto-fix: `pnpm lint:fix` (runs `oxlint --fix .`)

Configuration lives in `.oxlintrc.json`. `oxlint` is installed as a pnpm devDependency (or, in mise-based projects, declared in `mise.toml` and installed via `mise install`); in either case invoke it by bare name through pnpm. The `correctness`, `suspicious`, and `perf` categories are all hard errors; suppress a specific rule only with an inline named reason, never by turning a category off.
```

## Anti-patterns to avoid

- **Pinning with a caret.** `"oxlint": "^1.74.0"` drifts between machines and CI. Pin exactly, like Biome (P005).
- **Mixing install paths.** Pick one home for `oxlint`: devDependency or `mise.toml`, not both. Two homes is two versions that drift. (See P014.)
- **Putting oxlint and oxfmt on different `apps_v<N>` tags (mise path only).** They ship in one release; both `version` lines must match. (See P014.)
- **Leaving `suspicious`/`perf` at `"warn"` forever.** A warning that never fails CI is a warning that gets ignored. Resolve the findings and promote to `"error"`, or turn the category `"off"` deliberately — but `"warn"` is not a resting state.
- **Turning a category off to green a build.** Disabling `correctness`/`suspicious`/`perf` to make `pnpm lint` pass is a config regression. Fix the code, or carve out the specific rule with a named reason (the `__dirname`/`__filename` pattern above).
- **Suppressing a rule without a reason.** An override or inline-disable with no rationale is an unexplained hole. The JSONC format exists so every override can say why (same discipline as P007's markdownlint disables).
- **Linting generated output.** `dist/` and any bundled/vendored tree must be in `ignorePatterns`. Lint failures in generated code are noise that trains people to ignore the linter.
- **Introducing eslint/biome alongside oxlint.** oxlint owns JS/TS linting. A second linter produces conflicting rules and duplicated config. (oxfmt, P014, owns formatting — same project, different job.)
- **Omitting `env`.** Without `env.node: true` (or the matching runtime), oxlint flags legitimate globals as undefined and generates false positives. Set the env that matches your runtime.
