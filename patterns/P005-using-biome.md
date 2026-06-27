---
id: P005
name: using-biome
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/harness', 'github.com/capotej/pi-mise', 'github.com/capotej/pi-zsearch']
---

# Using Biome

Biome is the linter and formatter for TypeScript/JavaScript, pinned to an exact version in `devDependencies` and configured via a committed `biome.json`. The `$schema` version in `biome.json` must match the installed package version. Lint and format are exposed as pnpm scripts, and lint is gated in CI where CI exists.

## When to use

- Any TypeScript/JavaScript project that wants a single fast tool for both linting and formatting (Biome replaces eslint + prettier in one binary).
- Any project that already follows P004 (pnpm) — Biome is installed as a pnpm devDependency, not as a global or mise-managed tool.
- Any project where agents and CI should run the exact same lint/format pass before code is accepted.

## How it works

Four pieces work together: an exact-pinned devDependency, a committed `biome.json` with a version-matched schema, a set of ignore entries that exclude generated/cache directories, and the lint/format scripts that everyone invokes.

### Pin Biome to an exact version

`@biomejs/biome` goes in `devDependencies`, pinned to an exact version with no caret or range. Take the latest stable Biome release and pin that:

```json title="package.json"
{
  "devDependencies": {
    "@biomejs/biome": "1.9.4"
  }
}
```

```json title="package.json"
// Wrong — a range lets the installed version drift between machines and CI
{
  "devDependencies": {
    "@biomejs/biome": "^1.9.4"
  }
}
```

Biome ships breaking changes between major versions and occasionally between minors; an exact pin plus the committed `pnpm-lock.yaml` (P004) is what makes the lint result reproducible.

### Commit biome.json with a version-matched $schema

Biome is zero-config capable, but every reference repo commits an explicit `biome.json`. The critical rule: **the `$schema` URL version must match the installed `@biomejs/biome` version exactly.** When you bump Biome, you bump the schema URL in the same change:

```json title="biome.json"
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "linter": { "enabled": true, "rules": { "recommended": true } },
  "formatter": { "enabled": true }
}
```

A mismatched schema version produces silent config-drift: Biome validates against the schema it fetches, not the binary it runs, so unknown/new fields are ignored or mis-parsed. Treat the schema URL as a second copy of the version pin and keep them in lockstep.

### Ignore generated and cache directories

Every `biome.json` has a `files.ignore` array. Three entries are effectively mandatory given the rest of the convention set, because Biome would otherwise lint vendored/generated code and the package cache:

```json title="biome.json"
{
  "files": {
    "ignore": [
      "node_modules/",
      ".pnpm-store/",
      "dist/"
    ]
  }
}
```

- `node_modules/` and `.pnpm-store/` — the package caches from P004. `.pnpm-store/` is easy to forget because it only appears when the store is co-located with the project.
- `dist/` (or `bin/` for bundled CLI packages) — compiler output. Linting generated files just produces noise.

Add any other vendored or generated paths the project has (e.g. harness also ignores `opencode/`, `pi/`, `.harness/`, `.pi-lens/` — directories it vendors from other tools).

### Linter and formatter settings

The settings every reference repo shares:

```json title="biome.json"
{
  "linter": {
    "enabled": true,
    "rules": { "..." : "..." }
  },
  "formatter": {
    "enabled": true
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double"
    }
  }
}
```

Two valid approaches to `rules`, each suited to a different project shape:

**Recommended (start here).** Enable Biome's curated `recommended` set. Lowest friction; the right default for a service or app:

```json title="biome.json"
{
  "linter": {
    "enabled": true,
    "rules": { "recommended": true }
  }
}
```

**All rules with surgical opt-outs.** Enable `rules.all: true`, then turn off the few rules that conflict with the project's style. Used by library/extension packages where a stricter baseline is worth the opt-out maintenance:

```json title="biome.json"
{
  "linter": {
    "enabled": true,
    "rules": {
      "all": true,
      "correctness": {
        "noNodejsModules": "off",
        "noUndeclaredDependencies": "off"
      },
      "style": {
        "noDefaultExport": "off"
      }
    }
  }
}
```

The opt-outs shown above (`noNodejsModules`, `noUndeclaredDependencies`, `noDefaultExport`) recur across the pi-* extensions because they ship ESM modules that legitimately import Node built-ins, use peer dependencies, and export a default entry point. Pick the set your project actually needs — do not copy them blindly.

### Formatter style is per-project

Indent style (`space` vs `tab`), line width, and semicolons vary by repo and are a project decision, not part of the convention. The rule is only that they are declared once in `biome.json` and everyone lets Biome enforce them — no editor config fights.

### The lint and format script trio

Every repo exposes the same three pnpm scripts (invoked via pnpm per P004):

```json title="package.json"
{
  "scripts": {
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write ."
  }
}
```

- `lint` (`biome check .`) — runs linter + formatter in check mode; exits non-zero on violations. **This is the command CI and agents run.**
- `lint:fix` (`biome check --write .`) — lints and applies safe auto-fixes in place.
- `format` (`biome format --write .`) — applies formatting only.

Note harness composes `lint` from multiple sub-linters (`pnpm lint:ts && pnpm lint:md && ...`), where `lint:ts` is `biome check .`. Biome owns the JS/TS portion; the other linters (markdownlint, shellcheck, hadolint, actionlint) are orchestrated alongside it. Either a flat `lint` script or a composed one is fine — the rule is that running `pnpm lint` executes Biome.

### Gate lint in CI

Where the project has CI, the lint script is a required step:

```yaml title=".github/workflows/lint.yml"
steps:
  - uses: actions/checkout@<sha> # v5
  - uses: pnpm/action-setup@<sha> # v6
  - uses: actions/setup-node@<sha> # v6
    with:
      node-version: 24
      cache: pnpm
  - run: pnpm install --frozen-lockfile
  - run: pnpm lint
```

The pnpm setup follows P004; the actions are SHA-pinned per P002. Repos without CI (pi-mise, pi-zsearch) rely on the committed config plus developers/agents running `pnpm lint` locally.

## The AGENTS.md section

Add this so coding agents lint and format with Biome instead of inventing their own:

```markdown title="AGENTS.md"
## Linting and Formatting

This project uses **Biome** for both linting and formatting TypeScript/JavaScript. Do not introduce eslint, prettier, or editor-specific config.

- Check for violations: `pnpm lint` (runs `biome check .`)
- Auto-fix lint issues: `pnpm lint:fix` (runs `biome check --write .`)
- Format only: `pnpm format` (runs `biome format --write .`)

Configuration lives in `biome.json`. The `@biomejs/biome` version is pinned exactly in `package.json` devDependencies; the `$schema` URL in `biome.json` must match that version — if you bump Biome, bump both in the same change.

Before finishing any code change, run `pnpm lint`. If it reports violations you are allowed to fix, fix them with `pnpm lint:fix`; otherwise update your code. Do not edit `biome.json` rule settings to silence a violation without discussing it.
```

## Anti-patterns to avoid

- **Letting the `$schema` version drift from the installed Biome version.** The schema URL (`/schemas/1.9.4/`) and the devDependency version (`"1.9.4"`) are two copies of the same fact. Bumping one without the other causes silent validation/config drift.
- **Pinning Biome with a caret.** `"@biomejs/biome": "^1.9.4"` resolves differently across machines and CI, producing different lint results. Pin exactly, as with P002/P003/P004.
- **Forgetting `.pnpm-store/` in `files.ignore`.** With P004, the store can appear inside the project; Biome will happily lint thousands of vendored files. Always ignore `node_modules/`, `.pnpm-store/`, and build output (`dist/`, `bin/`).
- **Linting generated output.** `dist/`, `bin/`, or any compiled/vendored directory must be ignored. Lint failures in generated code are noise that trains people to ignore the linter.
- **Running Biome via a global install or as a mise tool.** Biome is a project devDependency invoked through pnpm (`biome ...` resolves to the pinned binary via `node_modules/.bin`). A global or mise-managed Biome is a different version and defeats the pin. This mirrors the P003 rule that package-managed tools are not mise tools.
- **Disabling rules to green a build.** Turning a rule off to make `pnpm lint` pass is a config regression. Fix the code, or discuss the opt-out deliberately (the pi-* extensions' opt-outs are intentional, not lint-suppression shortcuts).
- **Introducing eslint/prettier alongside Biome.** Biome does both jobs. Running a second linter/formatter produces conflicting rules and duplicated config.
- **Editor config files (`.editorconfig`, `.prettierrc`) that duplicate Biome's formatting decisions.** Keep formatting authority in `biome.json` only.

## Reference: biome.json across repos

### boldblackai/harness

```json title="package.json"
{
  "devDependencies": {
    "@biomejs/biome": "1.9.4"
  },
  "scripts": {
    "lint": "pnpm lint:ts && pnpm lint:md && pnpm lint:sh && pnpm lint:docker && pnpm lint:actions",
    "lint:ts": "biome check .",
    "format": "biome format --write ."
  }
}
```

```json title="biome.json"
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "files": {
    "ignore": ["bin/", "node_modules/", ".pnpm-store/", ".harness/", "opencode/", "pi/", ".pi-lens/"]
  },
  "linter": { "enabled": true, "rules": { "recommended": true } },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2 },
  "javascript": { "formatter": { "quoteStyle": "double" } }
}
```

A containerized agent runner. Uses the `recommended` rule set with `indentStyle: space`. Composes `lint` from six sub-linters; `lint:ts` is the Biome step. `lint.yml` runs `pnpm lint` in CI on every PR and push to main. Ignores the multiple vendored directories it pulls in (`opencode/`, `pi/`, `.pi-lens/`) in addition to the caches and build output.

### capotej/pi-mise

```json title="package.json"
{
  "devDependencies": { "@biomejs/biome": "1.9.4" },
  "scripts": {
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write ."
  }
}
```

```json title="biome.json"
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "files": { "ignore": [".pnpm-store", "node_modules", ".jj", ".git", "dist"] },
  "linter": {
    "enabled": true,
    "rules": {
      "all": true,
      "correctness": { "noNodejsModules": "off", "noUndeclaredDependencies": "off" },
      "style": { "noDefaultExport": "off" }
    }
  },
  "formatter": { "enabled": true, "indentStyle": "tab", "indentWidth": 2, "lineWidth": 100 },
  "javascript": { "formatter": { "quoteStyle": "double", "semicolons": "always" } }
}
```

A pi extension. Uses `rules.all: true` with three opt-outs suited to an ESM package that imports Node modules and exports a default entry. Enables `organizeImports`. No CI — Biome runs locally via the script trio.

### capotej/pi-zsearch

```json title="package.json"
{
  "devDependencies": { "@biomejs/biome": "1.9.4" },
  "scripts": {
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write ."
  }
}
```

```json title="biome.json"
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": { "enabled": true },
  "files": { "ignore": [".pnpm-store", "node_modules", ".jj", ".git", "dist"] },
  "linter": {
    "enabled": true,
    "rules": {
      "all": true,
      "correctness": { "noNodejsModules": "off", "noUndeclaredDependencies": "off" },
      "style": { "noDefaultExport": "off", "useNamingConvention": "off" }
    }
  },
  "formatter": { "enabled": true, "indentStyle": "tab", "indentWidth": 2, "lineWidth": 100 },
  "javascript": { "formatter": { "quoteStyle": "double", "semicolons": "always" } }
}
```

A pi extension (web_search/web_read tools). Same shape as pi-mise plus an additional `useNamingConvention: "off"` opt-out — the package's identifiers intentionally diverge from Biome's naming convention. Illustrates that `all: true` is meant to be paired with project-specific opt-outs, not copied wholesale.
