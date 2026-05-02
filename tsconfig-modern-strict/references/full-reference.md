# TypeScript Configuration Reference (ESNext + Maximum Strict)

This document is an LLM-oriented reference for setting up `tsconfig.json` with
modern defaults: `ESNext` targets, bundler-based module resolution, and the
strictest type-checking options that are practical for real projects.

**Default policy**

- Target the latest JavaScript syntax (`ESNext`).
- Enable every strict flag unless there is a concrete reason not to.
- Prefer `bundler` / `NodeNext` module resolution over the legacy `node` mode.
- Treat the type checker as a linter (`noEmit: true`) and let bundlers / tsc
  itself emit, depending on the use case.

---

## 1. Base Configuration

Use this as the starting point for application code that is consumed by a
bundler (Vite, webpack, Rspack, esbuild, Rolldown, etc.).

```json
{
  "compilerOptions": {
    /* Language and runtime */
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["esnext", "dom", "dom.iterable"],
    "jsx": "react-jsx",

    /* Emit */
    "noEmit": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,

    /* Interop */
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,

    /* Strictness (max) */
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Why each option

| Option | Reason |
| --- | --- |
| `target: "esnext"` | Emit modern syntax; downleveling is the bundler's job. |
| `module: "esnext"` | Preserve `import`/`export` so the bundler can tree-shake. |
| `moduleResolution: "bundler"` | Matches how Vite / webpack actually resolve modules; allows extensionless imports. |
| `lib` | Pin globals explicitly. Drop `dom` for Node-only code. |
| `noEmit: true` | tsc only type-checks; the bundler handles emit. |
| `isolatedModules: true` | Required when esbuild / SWC / Babel transpile per-file. |
| `verbatimModuleSyntax: true` | Forces explicit `import type` / `export type`; output mirrors source. |
| `allowImportingTsExtensions: true` | Permits `import x from "./foo.ts"`, useful with Vite, Deno, Bun. Pair with `noEmit: true`. |
| `esModuleInterop: true` | Allows `import express from "express"` against CJS modules. |
| `skipLibCheck: true` | Skips `.d.ts` checking inside `node_modules`. Massive speedup, near-zero downside. |
| `strict: true` | Enables eight sub-flags (see §3). |
| `noUncheckedIndexedAccess` | `arr[i]` and `obj[k]` become `T \| undefined`. Catches a huge class of bugs. |
| `exactOptionalPropertyTypes` | `{ x?: number }` rejects `{ x: undefined }`; `?:` means "may be absent", not "may be undefined". |
| `noImplicitOverride` | Subclass methods that override must say `override`. |

---

## 2. Module System: Picking the Right Trio

`target`, `module`, and `moduleResolution` must agree. Pick one row.

| Scenario | `target` | `module` | `moduleResolution` |
| --- | --- | --- | --- |
| Browser app via bundler (Vite/webpack/etc.) | `esnext` | `esnext` | `bundler` |
| Node.js native ESM (`"type": "module"`) | `esnext` | `nodenext` | `nodenext` |
| Node.js CommonJS (legacy) | `es2022` | `commonjs` | `node` |
| Dual-package library | `es2020` | `esnext` | `bundler` (build) |
| Deno / Bun script | `esnext` | `esnext` | `bundler` |

**Rule of thumb**: `bundler` is the default for application code. `nodenext` is
the default for code that runs directly under `node` without a bundler — note
that `nodenext` requires explicit `.js` extensions in imports
(`import { x } from "./foo.js"`, even when the source file is `foo.ts`).

---

## 3. Strict Mode: Full Map

`"strict": true` is a meta-flag. The individual flags it enables, plus the
extra strict flags that are *not* part of `strict`, are summarized here.

### Included in `strict: true`

| Flag | Effect |
| --- | --- |
| `noImplicitAny` | Disallow values whose type is implicitly `any`. |
| `strictNullChecks` | `null` and `undefined` are not assignable to other types. |
| `strictFunctionTypes` | Function parameters checked contravariantly. |
| `strictBindCallApply` | `bind`/`call`/`apply` are type-checked. |
| `strictPropertyInitialization` | Class fields must be initialized (or `!`-asserted, or optional). |
| `noImplicitThis` | `this` of type `any` is rejected. |
| `useUnknownInCatchVariables` | `catch (e)` infers `e: unknown`. |
| `alwaysStrict` | Parses in strict mode and emits `"use strict"`. |

### Strongly recommended additions (not in `strict`)

| Flag | Effect |
| --- | --- |
| `noUncheckedIndexedAccess` | Index access produces `T \| undefined`. |
| `exactOptionalPropertyTypes` | `?:` no longer accepts an explicit `undefined`. |
| `noImplicitOverride` | `override` keyword required for overrides. |
| `noImplicitReturns` | All code paths in a function must return a value (when one is returned anywhere). |
| `noFallthroughCasesInSwitch` | Empty non-terminal `case` blocks are an error. |
| `noUnusedLocals` | Unused locals are an error. |
| `noUnusedParameters` | Unused parameters are an error (prefix with `_` to opt out). |
| `allowUnreachableCode: false` | Unreachable code becomes an error, not a warning. |
| `allowUnusedLabels: false` | Unused labels become an error. |

### Optional but often worth it

| Flag | Effect |
| --- | --- |
| `noPropertyAccessFromIndexSignature` | Forces bracket access for index-signature properties. |
| `verbatimModuleSyntax` | Strict separation of value vs. type imports/exports. |
| `isolatedDeclarations` | (5.5+) Public APIs must have explicit types so a single file's `.d.ts` can be generated without cross-file inference. |
| `erasableSyntaxOnly` | (5.8+) Disallow TypeScript-only constructs (`enum`, parameter properties, namespaces) so the source can be type-stripped at runtime. |

### Anti-patterns to avoid

- `"strict": false` in new code — there is no scenario where this is the right
  default in 2026.
- Disabling `strictNullChecks` alone — it cascades into widespread bugs.
- Setting `noUnusedParameters: false` to silence interface implementations —
  use `_` prefix on the parameter instead, which is the official escape hatch.

---

## 4. File Selection: `include`, `exclude`, `files`

| Key | Behavior |
| --- | --- |
| `files` | Exact list. No globs. Highest precedence. |
| `include` | Globs; defaults to `**/*` when omitted. |
| `exclude` | Globs; subtracted from `include`. Does **not** affect `files`. |

`exclude` defaults to `node_modules`, `bower_components`, `jspm_packages`, and
the `outDir`. If you set `exclude` explicitly, the defaults are not applied,
so include `node_modules` again.

A file imported by an included file is automatically pulled in for type
checking even if it is outside `include`.

---

## 5. Use-Case Templates

### 5.1 Browser app + bundler (Vite, Next.js app code, etc.)

This is the §1 base configuration. Repeated here for convenience.

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "lib": ["esnext", "dom", "dom.iterable"],
    "jsx": "react-jsx",
    "noEmit": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src"]
}
```

### 5.2 Node.js ESM application

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["esnext"],
    "outDir": "./dist",
    "rootDir": "./src",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "dist"]
}
```

`package.json` must have `"type": "module"`. Imports must include the `.js`
extension even though the source file is `.ts`:

```ts
import { foo } from "./foo.js"; // resolves ./foo.ts at compile time
```

### 5.3 Publishable library (separate JS and `.d.ts` output)

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "rootDir": "./src",
    "outDir": "./dist",
    "declarationDir": "./dist-types",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "allowImportingTsExtensions": false,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts"]
}
```

Library publishing notes:

- For distribution builds, set `allowImportingTsExtensions: false` and remove
  `.ts` extensions from imports — consumers won't have your `.ts` files.
- Use `declarationDir` to keep `.d.ts` files separate from `.js` files. This
  lets bundlers (e.g. tsdown, tsup) emit JS while tsc emits only types.
- Consider `isolatedDeclarations` (TS 5.5+) so external tools can generate
  `.d.ts` from a single file without whole-program inference.

### 5.4 Monorepo with project references

Root `tsconfig.json`:

```json
{
  "files": [],
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/utils" },
    { "path": "./packages/app" }
  ]
}
```

Each package needs `"composite": true` and a `references` array pointing to
its dependencies:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src/**/*"],
  "references": [{ "path": "../core" }]
}
```

Build with `tsc --build` (or `tsc -b`). This produces `.tsbuildinfo` files
that enable incremental rebuilds across the whole workspace.

---

## 6. `package.json` Companion: Library Exports

The `tsconfig.json` is only half the story for libraries. The package must
declare its entry points so consumers see both the JS and the types.

```json
{
  "name": "@your-scope/package-name",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist-types/index.d.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    },
    "./utils": {
      "types": "./dist-types/utils.d.ts",
      "import": "./dist/utils.js",
      "default": "./dist/utils.js"
    },
    "./package.json": "./package.json"
  },
  "main": "./dist/index.js",
  "types": "./dist-types/index.d.ts",
  "files": ["dist", "dist-types", "!**/*.test.*", "!**/*.spec.*"],
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch",
    "clean": "rm -rf dist dist-types",
    "prepublishOnly": "pnpm run clean && pnpm run build"
  }
}
```

Key rules for `exports`:

- `types` **must come first** in each conditional object. Resolvers stop at
  the first match.
- Always include `"./package.json": "./package.json"` so tools that read it
  (linters, bundlers) keep working.
- Use `main` and top-level `types` as fallbacks for older tooling.

### Subpath patterns

For multi-entry libraries, glob the entries instead of listing them:

```json
{
  "exports": {
    ".": {
      "types": "./dist-types/index.d.ts",
      "import": "./dist/index.js"
    },
    "./*": {
      "types": "./dist-types/*.d.ts",
      "import": "./dist/*.js"
    },
    "./components": {
      "types": "./dist-types/components/index.d.ts",
      "import": "./dist/components/index.js"
    },
    "./components/*": {
      "types": "./dist-types/components/*.d.ts",
      "import": "./dist/components/*.js"
    }
  }
}
```

The `*` wildcard captures the subpath and reuses it on the target side. The
files referenced must actually exist after build.

---

## 7. Path Aliases

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "~/*": ["src/*"]
    }
  }
}
```

Caveats:

- `paths` only affects type checking. The bundler / runtime needs the same
  rules configured separately (Vite: `resolve.alias`; webpack: `resolve.alias`;
  Node: import maps or a loader).
- With `moduleResolution: "bundler"`, `baseUrl` is no longer required for
  `paths` to work.
- Prefer one alias style per project. Mixing `@/` and `~/` is a smell.

---

## 8. Performance Tuning

| Option | Effect |
| --- | --- |
| `incremental: true` | Emit `.tsbuildinfo` for fast subsequent type-checks. |
| `composite: true` | Required for project references; implies `incremental`. |
| `tsBuildInfoFile` | Custom location for `.tsbuildinfo` (default: alongside output). |
| `skipLibCheck: true` | Skip `.d.ts` inside dependencies. Almost always on. |
| `assumeChangesOnlyAffectDirectDependencies` | (Watch mode) only re-check direct dependents. Faster but less safe. |

For large projects:

1. Enable `skipLibCheck`.
2. Use project references to split the build graph.
3. Run `tsc --noEmit --incremental` in CI to cache type-check results.
4. Keep `include` narrow; don't `include: ["**/*"]` from the repo root.

Diagnostic commands:

```sh
tsc --showConfig          # print the fully resolved config (after extends)
tsc --traceResolution     # log every module resolution step
tsc --extendedDiagnostics # detailed phase timings
tsc --listFiles           # list every file in the program
```

---

## 9. Sharing Configuration with `extends`

Place common settings in `tsconfig.base.json` at the repo root:

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

Then in each package:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "./dist", "rootDir": "./src" },
  "include": ["src"]
}
```

`extends` accepts an array (TS 5.0+) for layered presets:

```json
{ "extends": ["@tsconfig/strictest/tsconfig", "./tsconfig.base.json"] }
```

Notable community presets on npm:

- `@tsconfig/strictest` — every reasonable strict flag turned on.
- `@tsconfig/node20`, `@tsconfig/node22` — runtime-targeted defaults.
- `@tsconfig/recommended` — minimal modern baseline.

---

## 10. Common Pitfalls

**Mixed module / resolution settings.** `module: "commonjs"` with
`moduleResolution: "bundler"` is invalid; the resolver expects matching ESM
output. Use the table in §2.

**`paths` works in tsc but breaks at runtime.** TypeScript only rewrites
imports if `module` is CommonJS *and* it's emitting. With `noEmit: true`,
`paths` is a type-only hint; configure the bundler too.

**`enum` and `const enum` under `isolatedModules`.** `const enum` is rejected
because per-file transpilers can't inline its values. Use a `const` object
with `as const` instead.

**`verbatimModuleSyntax` rejects mixed imports.** Split them:

```ts
// Wrong
import { foo, type Bar } from "./mod"; // OK actually — syntax is fine
import Foo, { type Bar } from "./mod"; // also OK

// Wrong: implicit type-only re-export
export { Bar } from "./mod"; // error if Bar is type-only
export type { Bar } from "./mod"; // correct
```

**`exactOptionalPropertyTypes` breaks spread patterns.** Code like
`{ ...defaults, x: maybeUndefined }` no longer satisfies `{ x?: number }`
because the property is now present-with-undefined. Either omit the key or
widen the type to `{ x?: number | undefined }`.

**Stale `.tsbuildinfo`.** When changing `compilerOptions`, delete cached
`.tsbuildinfo` files; tsc does not always invalidate them automatically.

**`skipLibCheck: false` in a polyglot dependency tree.** A single broken
`.d.ts` file in `node_modules` blocks the whole build. Keep `skipLibCheck`
on unless you have a specific reason.

---

## 11. Migration Cheatsheet (Loose → Strict)

For an existing codebase, enable flags in this order to minimize churn:

1. `strict: true` — fix `null`/`undefined` and implicit `any`.
2. `noUnusedLocals` + `noUnusedParameters` — easy wins.
3. `noImplicitReturns` + `noFallthroughCasesInSwitch` — control-flow checks.
4. `noUncheckedIndexedAccess` — many sites, mostly `arr[i]!` or guards.
5. `exactOptionalPropertyTypes` — last; touches API shapes.
6. `verbatimModuleSyntax` — also touches every import file.

Run `tsc --noEmit` after each step. Commit between steps.

---

## 12. Quick Decision Tree

```
Building an app?
├── Browser via bundler        → §5.1  (esnext + bundler + noEmit)
├── Node.js ESM                → §5.2  (esnext + nodenext)
└── Node.js CJS (legacy)       → adapt §5.2 with module/moduleResolution = node
Building a library?
├── Single package             → §5.3  (declaration + declarationMap)
└── Monorepo                   → §5.4  (project references + composite)
Need fast incremental builds?  → §8    (incremental, references, skipLibCheck)
Need to share config?          → §9    (extends, presets)
```
