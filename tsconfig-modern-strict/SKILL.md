---
name: tsconfig-modern-strict
description: Use whenever a tsconfig.json is being created, edited, reviewed, troubleshot, or even just discussed. Triggers on any mention of "tsconfig", "TypeScript config", strict mode flags, module resolution, target/module decisions, project references, monorepo TS setup, library publishing config, path aliases, or "なんでこの型エラーが出る/出ない" debates that turn out to be config-shaped. Apply this skill even when the user only asks about one option — it encodes a coherent ESNext + maximum-strict policy that should drive the rest of the config to stay consistent. Default to this skill instead of free-recall whenever the answer involves a tsconfig field.
---

# tsconfig-modern-strict

Opinionated runbook for producing TypeScript configs that target modern toolchains with the strictest practical type checking.

**Standing policy** (override only with an explicit, stated reason):

- `target: "esnext"`. Downleveling is the bundler/runtime's job, not tsc's.
- `strict: true` plus every recommended add-on flag in §3.
- `moduleResolution: "bundler"` for app code, `"nodenext"` for native Node.
- `skipLibCheck: true`. Always.
- `isolatedModules: true` and `verbatimModuleSyntax: true` for any code touched by esbuild/SWC/Babel/Vite.

When the user proposes loosening any of these, ask why before agreeing. Most of the time the request comes from copy-pasted legacy advice rather than a real constraint.

---

## 1. Pick the use case first (decision tree)

```
Is tsc emitting JS?
├── No, a bundler emits  ─────────────────► §4.1  Browser app + bundler
└── Yes, tsc emits
    ├── Running on Node directly
    │   ├── ESM ("type": "module")  ─────► §4.2  Node ESM app
    │   └── CommonJS (legacy only)  ─────► adapt §4.2 (see notes)
    └── Publishing to npm
        ├── Single package           ────► §4.3  Library
        └── Monorepo                 ────► §4.4  Project references
```

Resolve the use case **before** writing or modifying a tsconfig. The four templates in §4 differ in ways that don't compose cleanly — picking wrong forces a rewrite, not a patch.

---

## 2. The trio that must agree: `target` / `module` / `moduleResolution`

These three options are co-dependent. Use this table; do not improvise.

| Scenario                              | `target`  | `module`     | `moduleResolution` |
| ------------------------------------- | --------- | ------------ | ------------------ |
| Browser app via bundler               | `esnext`  | `esnext`     | `bundler`          |
| Node.js native ESM                    | `esnext`  | `nodenext`   | `nodenext`         |
| Node.js CommonJS (legacy)             | `es2022`  | `commonjs`   | `node`             |
| Publishable library (build via tsc)   | `es2020`+ | `esnext`     | `bundler`          |
| Deno / Bun script                     | `esnext`  | `esnext`     | `bundler`          |

Common mismatches that produce confusing errors:

- `module: "commonjs"` + `moduleResolution: "bundler"` — invalid.
- `module: "esnext"` + `moduleResolution: "node"` — legacy resolver, won't see `exports` in `package.json`.
- `nodenext` + extensionless imports — Node ESM requires `.js` on relative imports even when the source is `.ts`.

---

## 3. Strict policy (apply to every template)

### Always on (`strict: true` covers these eight)

`noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `useUnknownInCatchVariables`, `alwaysStrict`.

### Always add on top

These are not part of `strict` but should be enabled by default. Document any omission in the config with a comment.

| Flag | What it catches |
| --- | --- |
| `noUncheckedIndexedAccess` | `arr[i]` and `obj[k]` become `T \| undefined` — kills a huge class of bugs. |
| `exactOptionalPropertyTypes` | `?:` no longer accepts an explicit `undefined`. |
| `noImplicitOverride` | Subclass overrides must say `override`. |
| `noImplicitReturns` | All branches return when any branch does. |
| `noFallthroughCasesInSwitch` | No silent `case` fallthrough. |
| `noUnusedLocals` | Unused locals are errors. |
| `noUnusedParameters` | Unused parameters are errors. Prefix with `_` to opt out per parameter — never disable globally. |
| `allowUnreachableCode: false` | Unreachable code is an error, not a warning. |
| `allowUnusedLabels: false` | Same for labels. |

### Add when the toolchain supports it

| Flag | Add when |
| --- | --- |
| `verbatimModuleSyntax` | Any non-tsc transpiler in the pipeline (esbuild, SWC, Babel, Vite). |
| `isolatedModules` | Same as above. |
| `noPropertyAccessFromIndexSignature` | The codebase uses index signatures and you want explicit bracket access. |
| `isolatedDeclarations` (5.5+) | Publishing a library and using a non-tsc declaration emitter. |
| `erasableSyntaxOnly` (5.8+) | Source must be runnable after pure type-stripping (Node `--experimental-strip-types`, Deno, Bun). Disallows `enum`, parameter properties, namespaces. |

### Refuse to disable

Pushback list — if the user wants any of these turned off, ask what concrete problem they are trying to solve before agreeing.

- `strict` (or any of its eight sub-flags individually)
- `skipLibCheck` (turning it off blocks builds on a single broken `.d.ts` in `node_modules`)
- `noUnusedParameters` globally (the `_param` escape hatch is the right answer)

---

## 4. Templates

Copy the matching one as a starting point, then layer project-specific tweaks (paths, lib, jsx) on top. **Do not delete strict flags to "simplify"**.

### 4.1 Browser app + bundler (Vite, Next.js app code, Rspack, etc.)

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
    "noUnusedParameters": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false
  },
  "include": ["src"]
}
```

Tweaks:
- Drop `"dom"` and `"dom.iterable"` from `lib` for non-browser code.
- Change `jsx` to `"preserve"` + set `jsxImportSource` for Preact/Solid.
- Remove `allowImportingTsExtensions` if the bundler can't handle it.

### 4.2 Node.js ESM application

`package.json` must have `"type": "module"`. Relative imports must include `.js` even when the source is `.ts`.

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["esnext"],

    "outDir": "./dist",
    "rootDir": "./src",
    "sourceMap": true,

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

For legacy CommonJS Node: change to `"module": "commonjs"`, `"moduleResolution": "node"`, drop `verbatimModuleSyntax`, and remove `"type": "module"` from `package.json`.

### 4.3 Publishable library (separate JS and `.d.ts` output)

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

When publishing, also configure `package.json` `exports` — `types` field **first** in each conditional block, fallback `main`/`types` at top level for old tooling. See `references/full-reference.md` §6.

### 4.4 Monorepo with project references

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

Each package extends the shared base and sets `composite`:

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

Build with `tsc --build` (or `tsc -b`). Don't use plain `tsc` against the root — references won't resolve.

---

## 5. Path aliases

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

Rules:

- `paths` only affects type checking. The bundler/runtime needs the same alias configured separately (Vite `resolve.alias`, webpack `resolve.alias`, Node import maps, etc.).
- With `moduleResolution: "bundler"`, `baseUrl` is optional.
- One alias style per project. Don't mix `@/` and `~/`.

---

## 6. Pitfalls — apply as rules, not theory

When any of these come up, recognize the symptom and fix the cause directly.

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Cannot find module './foo'` in Node ESM | Missing `.js` in import | Add `.js` extension (yes, on `.ts` source). |
| `paths` works in tsc, fails at runtime | tsc doesn't rewrite imports under `noEmit` | Configure the bundler's alias too. |
| `const enum` rejected | `isolatedModules: true` | Replace with `as const` object. |
| Spread breaks `exactOptionalPropertyTypes` | `{...x, y: maybeUndef}` makes `y` present-with-undefined | Conditionally include the key, or widen to `y?: T \| undefined`. |
| Stale type errors after config change | `.tsbuildinfo` cache | Delete `.tsbuildinfo` and rebuild. |
| `import type` enforcement breaks builds | `verbatimModuleSyntax` + value/type mix | Split into two imports, or use `import { type X, y }`. |
| Single broken `.d.ts` blocks build | `skipLibCheck: false` | Re-enable `skipLibCheck`. |
| `exports` resolves wrong file for types | `types` not first in conditional object | Move `types` to the top of each conditional block. |

---

## 7. Diagnostic commands

When debugging a config, run these instead of guessing:

```sh
tsc --showConfig            # final resolved config after all `extends`
tsc --traceResolution       # every module-resolution step (huge output, pipe to a file)
tsc --extendedDiagnostics   # phase timings for performance work
tsc --listFiles             # every file the program is compiling
```

`tsc --showConfig` is the single most valuable debugging tool and underused. Run it first whenever a setting "isn't working".

---

## 8. Migration order (loose → strict)

When tightening an existing project, enable flags in this order to minimize churn. Run `tsc --noEmit` after each step. Commit between steps.

1. `strict: true`
2. `noUnusedLocals` + `noUnusedParameters`
3. `noImplicitReturns` + `noFallthroughCasesInSwitch`
4. `noUncheckedIndexedAccess`
5. `exactOptionalPropertyTypes`
6. `verbatimModuleSyntax`

Do **not** enable everything at once on an existing codebase — the error volume becomes unreadable and bugfixes get mixed with churn.

---

## 9. Sharing config across packages

Place common settings in `tsconfig.base.json` at the repo root and have each package `extends` it:

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

Useful community presets: `@tsconfig/strictest`, `@tsconfig/node20`, `@tsconfig/node22`.

---

## When to consult `references/full-reference.md`

The reference file expands on what's in this skill with prose explanations of each option, the full strict-flag breakdown, library `package.json` `exports` patterns including subpath wildcards, and a longer pitfalls section. Pull it in when:

- The user asks **why** an option behaves a certain way (this skill says "do X"; the reference says "because Y").
- A `package.json` `exports` field needs to be authored or audited beyond the one-liner in §4.3.
- Performance tuning is the goal (`incremental`, `composite`, project-reference graph design).
- The user is on TypeScript ≤ 4.x and modern flag names don't apply.
