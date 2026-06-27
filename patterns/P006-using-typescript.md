---
id: P006
name: using-typescript
date: 2026-06-27
author: Julio Capote
used_in: ['github.com/boldblackai/harness', 'github.com/capotej/pi-zsearch']
---

# Using TypeScript

TypeScript is compiled with `tsc`, configured via a committed `tsconfig.json` with `strict` mode on, a single `src/` source root, and declaration emit. Build and typecheck go through pnpm scripts; typecheck is gated in CI where CI exists. `tsc` is the only compiler — no babel, no swc, no custom pipeline.

## When to use

- Any Node/TypeScript project that wants compile-time type safety and `.d.ts` output for consumers.
- Any project already following P004 (pnpm) and P005 (Biome) — TypeScript rounds out the JS/TS toolchain as the compiler.
- Any project where a single `tsc` invocation should both typecheck and emit, with no extra build steps.

## How it works

### Pin the toolchain, not the compiler

`tsc` ships with the `typescript` package; it is NOT a mise tool. It's a devDependency, pinned via the lockfile (P004), invoked through pnpm. Take the latest stable TypeScript and pin it:

```json title="package.json"
{
  "devDependencies": {
    "typescript": "^6.0.2",
    "@types/node": "^25.6.0"
  }
}
```

```json title="package.json"
// Wrong — installing tsc globally or via mise. The project's pinned tsc
// in node_modules/.bin is the one that runs; a global one is a different version.
```

Node's types (`@types/node`) go alongside it, with a version matching the Node runtime the project targets.

### strict: true is non-negotiable

Every tsconfig enables `strict`. This turns on the full strict family (`noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, etc.) as a group. Do not enable strict and then weaken individual strict flags, and do not reach for `strict: false` to make errors go away:

```json title="tsconfig.json"
{
  "compilerOptions": {
    "strict": true
  }
}
```

If the codebase has pre-existing type holes, fix them; don't relax the compiler.

### Single source root, single emit dir

All TypeScript lives under `src/`; everything else compiles to a single `outDir`. This is enforced by three fields that must agree:

```json title="tsconfig.json"
{
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src/**/*"]
}
```

- `rootDir: "src"` — the compiler root; preserves directory structure in the output.
- `include: ["src/**/*"]` — only files under `src/` are part of the compilation.
- `outDir` — the emit destination (`dist` or `bin`).

`exclude` must contain `node_modules` and the `outDir` itself (so the compiler never re-reads its own emitted `.js`/`.d.ts`):

```json title="tsconfig.json"
{
  "exclude": ["node_modules", "dist"]
}
```

### Emit declarations

`declaration: true` emits `.d.ts` next to the `.js`. This is mandatory because the package is meant to be consumed — the `files` allowlist in `package.json` ships the output, and consumers need the types:

```json title="tsconfig.json"
{
  "compilerOptions": {
    "declaration": true
  }
}
```

### Explicit types array

Both tsconfigs set `types: ["node"]` rather than letting TypeScript auto-include every package found under `node_modules/@types`. This keeps the global type surface intentional — only `@types/node` is pulled into every file automatically; other ambient types must be imported explicitly:

```json title="tsconfig.json"
{
  "compilerOptions": {
    "types": ["node"]
  }
}
```

### Standard interop and speed flags

```json title="tsconfig.json"
{
  "compilerOptions": {
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

- `esModuleInterop: true` — correct default-import interop with CommonJS modules.
- `skipLibCheck: true` — skip type-checking the `.d.ts` of dependencies. Faster builds; the project's own code is still fully checked.

### target, module, and moduleResolution are per-project

These three are chosen by the deployment target, and that's the one place the two reference repos legitimately differ:

| Repo | target | module | moduleResolution | outDir |
| --- | --- | --- | --- | --- |
| harness | ES2020 | CommonJS | bundler | bin |
| pi-zsearch | ES2022 | NodeNext | NodeNext | dist |

The rule is not "use ES2022" or "use NodeNext" — it's: **pick the lowest target/module that covers your runtime, and make `module` and `moduleResolution` agree** (e.g. `NodeNext`/`NodeNext`, or a CJS module with a compatible resolution). Mismatched module/resolution pairs cause import-resolution bugs that are hard to trace.

### The build, prepare, and typecheck scripts

```json title="package.json"
{
  "scripts": {
    "build": "tsc",
    "prepare": "tsc"
  }
}
```

- `build` — compiles `src/` to `outDir`. The canonical build command.
- `prepare` — runs `tsc` on `pnpm install`. This matters for consumers who install from a git URL: they get a built package without a separate build step.

For typecheck-only (no emit), invoke `tsc --noEmit` directly. harness exposes this in CI rather than as a script:

```json title="package.json"
{
  "scripts": {
    "build": "tsc && cp -r src/seccomp-profiles bin/seccomp-profiles",
    "prepare": "tsc"
  }
}
```

harness composes its `build` with a `cp` of non-TS assets (seccomp profiles) — `tsc` emits the JS/declarations, the extra step copies files `tsc` doesn't know about. Either a plain `tsc` build or a composed one is fine; the rule is that `build` starts with `tsc`.

### Gate typecheck in CI

Where CI exists, `tsc --noEmit` runs as a step separate from (and before) the build, so type errors fail the pipeline without producing artifacts:

```yaml title=".github/workflows/e2e.yml"
- run: pnpm install --frozen-lockfile
- run: pnpm exec tsc --noEmit
- run: pnpm build
- run: pnpm test:coverage
```

Repos without CI (pi-zsearch) rely on the committed tsconfig and developers/agents running `pnpm build` locally.

## The AGENTS.md section

```markdown title="AGENTS.md"
## TypeScript

This project is written in TypeScript and compiled with `tsc` (no babel/swc). Configuration lives in `tsconfig.json`; `strict` mode is on — do not disable it or weaken individual strict flags.

- Build (compile + emit declarations): `pnpm build`
- Typecheck only, no emit: `pnpm exec tsc --noEmit`
- Source lives in `src/` and compiles to the `outDir` named in tsconfig.json; do not add TypeScript outside `src/`.

When adding a dependency, also add its `@types/*` package if the dependency doesn't ship its own types. When changing compiler options, keep `module` and `moduleResolution` consistent with each other, and do not lower `target` below what the runtime supports.
```

## Anti-patterns to avoid

- **`strict: false` or partial strict with individual flags off.** The strict family is the baseline. Disabling it (or turning off `strictNullChecks` etc. individually) reintroduces the type holes TypeScript exists to catch.
- **Installing `typescript` or `tsc` globally / via mise.** `tsc` is a devDependency invoked through pnpm; it resolves to the pinned binary in `node_modules/.bin`. A global or mise-managed tsc is a different version and defeats the lockfile. Mirrors the P003/P005 rule that package-managed tools are not mise tools.
- **Source files outside `src/`.** With `rootDir: "src"` and `include: ["src/**/*"]`, TypeScript outside `src/` is either ignored or breaks `rootDir`. Keep all `.ts` under `src/`.
- **Omitting the `outDir` from `exclude`.** If `outDir` isn't excluded, `tsc --watch` and incremental builds can re-read emitted `.d.ts`/`.js`, causing stale-type errors. Always exclude `node_modules` and the emit dir.
- **`module` / `moduleResolution` mismatches.** e.g. `module: ESNext` with `moduleResolution: node`. The pair must be consistent (NodeNext/NodeNext, or a CJS module with a compatible resolution). Mismatches cause import-resolution failures that surface far from the cause.
- **Letting `types` auto-resolve from `node_modules/@types`.** Without an explicit `types` array, every `@types/*` package installed becomes globally ambient. Set `types: ["node"]` (plus only what's intentionally global) and import other types explicitly.
- **Adding babel/swc/esbuild on top of `tsc`.** The project compiles with `tsc` alone. A second transpiler introduces divergent emit and a second config to maintain.
- **A `build` script that doesn't start with `tsc`.** If `build` shells out to a bundler or a custom script that bypasses typecheck, type errors stop failing the build. `build` must invoke `tsc` (plain or composed with asset copies).
- **No `prepare` script.** Without `prepare: "tsc"`, consumers installing from a git URL get an unbuilt package. `prepare` runs on install and ships a ready-to-use package.

## Reference: tsconfig.json across repos

### boldblackai/harness

```json title="package.json"
{
  "scripts": {
    "build": "tsc && cp -r src/seccomp-profiles bin/seccomp-profiles",
    "prepare": "tsc"
  },
  "devDependencies": {
    "typescript": "^6.0.2",
    "@types/node": "^25.6.0"
  }
}
```

```json title="tsconfig.json"
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "bin",
    "rootDir": "src",
    "declaration": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

A containerized agent runner that ships a CLI (`bin/harness.js`). Targets ES2020/CommonJS for broad Node compatibility, emits to `bin/`. Build composes `tsc` with a `cp` of seccomp profiles that aren't TypeScript. `e2e.yml` runs `pnpm exec tsc --noEmit` before `pnpm build` in CI.

### capotej/pi-zsearch

```json title="package.json"
{
  "type": "module",
  "scripts": {
    "build": "tsc",
    "prepare": "tsc"
  },
  "devDependencies": {
    "typescript": "^6.0.2",
    "@types/node": "^22.0.0"
  }
}
```

```json title="tsconfig.json"
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

A pi extension (ESM package, `"type": "module"`). Targets ES2022 with the NodeNext module pair to match modern Node ESM semantics, emits to `dist/`. Plain `tsc` build with no asset copies. No CI — typecheck relies on `pnpm build` locally.
