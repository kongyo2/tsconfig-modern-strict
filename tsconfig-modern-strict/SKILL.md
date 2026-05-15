---
name: tsconfig-modern-strict
description: 'Use when creating, editing, or troubleshooting `tsconfig.json` — strict flag selection, `target`/`module`/`moduleResolution` trio, project references, monorepo TS setup, library publishing, path aliases. Trigger on `tsconfig.json` / `tsconfig.base.json` / `tsconfig.test.json` files, `Cannot find module ''./foo''` errors in Node ESM, `exactOptionalPropertyTypes` spread failures, `.tsbuildinfo` cache staleness, `tsc --showConfig` / `tsc --build` / `tsc --traceResolution` topics, or "なんでこの型エラーが出る/出ない" debates that turn out to be config-shaped — even when the user does not name "tsconfig" or "TypeScript config". Apply even when the user asks about a single flag: this skill encodes a coherent ESNext + maximum-strict policy. Default to this skill instead of free-recall whenever the answer involves a tsconfig field. Not for: pure TS syntax/type-system questions, `jsconfig.json`, or non-tsc build tooling (Vite/webpack/Rollup configs themselves).'
---

# tsconfig-modern-strict

TypeScript 5.x 向けに、ESNext を狙い、最大限 strict な型検査を運用するための tsconfig 運用書。判断・最短パス・落とし穴をこの順で並べる。

---

## いつ使うか

- 新しい TypeScript プロジェクトで `tsconfig.json` を作る
- 既存の `tsconfig.json` を strict 化・モダン化する
- 「なぜこの型エラーが出る/出ない」が config 起因と疑う
- monorepo の project references、library 公開、path 別名で迷う
- 単一の flag を変える相談（一つ変えると他に影響することが多いため、ここに来る価値がある）

**Standing policy**（覆すには明示的な理由を述べる）:

- `target: "esnext"`. Downlevel は bundler / runtime の仕事、tsc の仕事ではない。
- `strict: true` + §3 の追加 strict flag 全部。
- `moduleResolution: "bundler"`（アプリ）／`"nodenext"`（ネイティブ Node）。
- `skipLibCheck: true`. 常時。
- `isolatedModules: true` + `verbatimModuleSyntax: true`（esbuild / SWC / Babel / Vite を通すコード全て）。

緩めたいリクエストが来たら、まず理由を聞く。多くは legacy のコピペアドバイスが理由で、実際の制約ではない。

---

## 使わない場面

- 純 JS プロジェクトの `jsconfig.json`（trigger するが内容は当てはまらない）
- TypeScript の構文・型システム自体の質問（このスキルは config 専属）
- Vite / webpack / Rollup の config 本体（tsc 側からの設定だけが対象）
- runtime としての `tsx` / `ts-node` の挙動チューニング（`tsconfig.json` の `ts-node` セクションは別物）

---

## クイックスタート（最頻ケース: Vite / Next.js などの browser app）

迷ったらまずこれ。`src/` ディレクトリ前提。

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

これが当てはまらない場合（Node 直接実行、ライブラリ公開、monorepo）は §1 で分岐して §4 のテンプレへ。

---

## §1. Pick the use case first (decision tree)

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

テストだけ別 config にしたいときは §4.5（`tsconfig.test.json`）。

---

## §2. The trio that must agree: `target` / `module` / `moduleResolution`

これらは独立に決められない。表の通りに揃える。

| Scenario                              | `target`   | `module`            | `moduleResolution` |
| ------------------------------------- | ---------- | ------------------- | ------------------ |
| Browser app via bundler               | `esnext`   | `esnext`/`preserve` | `bundler`          |
| Node.js native ESM                    | `esnext`   | `nodenext`          | `nodenext`         |
| Node.js CommonJS (legacy)             | `es2022`   | `commonjs`          | `node`             |
| Publishable library (build via tsc)   | `es2022`+  | `esnext`            | `bundler`          |
| Deno (deno check / dnt 経由)         | `esnext`   | `esnext`            | `nodenext`         |
| Bun script                            | `esnext`   | `esnext`            | `bundler`          |

- `module: "preserve"` は TS 5.4+。`bundler` 系では `esnext` の代替として推奨。CJS/ESM の判断を bundler に完全委譲し、tsc は何も書き換えない。`verbatimModuleSyntax` との相性が良い。
- Library の `target` は consumer の Node 互換性で決める。Node 18 LTS なら `es2022`、Node 20 LTS なら `es2023`。`esnext` は consumer 側の不確実性を増すので避ける。

Common mismatches that produce confusing errors:

- `module: "commonjs"` + `moduleResolution: "bundler"` — invalid.
- `module: "esnext"` + `moduleResolution: "node"` — legacy resolver、`package.json` の `exports` を見ない。
- `nodenext` + extensionless imports — Node ESM は相対 import に `.js` 必須（`.ts` ソースであっても）。TS 5.7+ なら `--rewriteRelativeImportExtensions` で書き換え可（§4.2 参照）。

詳細・なぜ: full-reference.md §2 (flag-by-flag rationale)。

---

## §3. Strict policy (apply to every template)

### Always on (`strict: true` covers these eight)

`noImplicitAny`, `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `useUnknownInCatchVariables`, `alwaysStrict`.

### Always add on top

`strict` には入らないが、デフォルトで有効化すべき。落とすなら理由をコメントで残す。

| Flag | What it catches |
| --- | --- |
| `noUncheckedIndexedAccess` | `arr[i]` / `obj[k]` が `T \| undefined` になる。バグの大量検出。 |
| `exactOptionalPropertyTypes` | `?:` が明示的 `undefined` を拒否する。 |
| `noImplicitOverride` | サブクラスの override に `override` キーワードが必須。 |
| `noImplicitReturns` | どこかで return があれば、全パスで return。 |
| `noFallthroughCasesInSwitch` | サイレント `case` fallthrough 禁止。 |
| `noUnusedLocals` | 未使用ローカルはエラー。 |
| `noUnusedParameters` | 未使用引数はエラー。`_` プレフィックスで個別 opt-out。グローバルに無効化しない。 |
| `allowUnreachableCode: false` | 到達不能コードはエラー（警告ではなく）。 |
| `allowUnusedLabels: false` | ラベルも同様。 |

### Add when the toolchain supports it

| Flag | Add when |
| --- | --- |
| `verbatimModuleSyntax` | tsc 以外の transpiler を通すとき (esbuild, SWC, Babel, Vite)。 |
| `isolatedModules` | 同上。 |
| `noPropertyAccessFromIndexSignature` | index signature を多用していて、明示的に `["x"]` 経由でアクセスさせたいとき。 |
| `noUncheckedSideEffectImports` (5.6+) | `import "./side-effects.js"` のタイポを検出。 |
| `isolatedDeclarations` (5.5+) | ライブラリ公開で、非 tsc な `.d.ts` 生成ツールを使うとき。 |
| `erasableSyntaxOnly` (5.8+) | ソースを純粋な type-stripping で実行する場合 (Node `--experimental-strip-types`, Deno, Bun)。`enum`, parameter properties, namespace を禁止。 |

### Refuse to disable

これらを切りたいと言われたら、まず具体的に何が困っているか聞く。

- `strict`（およびその 8 サブフラグ個別）
- `skipLibCheck`（切ると `node_modules` 内の壊れた `.d.ts` 一個でビルドが止まる）
- `noUnusedParameters` をグローバルに（`_param` の escape hatch が正解）

詳細・なぜ: full-reference.md §2。

---

## §4. Templates

該当する一個をコピーし、project 固有の差分（`paths`, `lib`, `jsx`）を上に乗せる。**strict flag を「簡単にするため」に削らない**。

### §4.1 Browser app + bundler (Vite, Next.js, Rspack, etc.)

最頻ケース。冒頭のクイックスタートと同一。Tweaks:

- 非ブラウザコードなら `lib` から `"dom"` / `"dom.iterable"` を落とす。
- Preact / Solid なら `jsx: "preserve"` + `jsxImportSource` を別途設定。
- bundler が `.ts` 拡張子の import を扱えないなら `allowImportingTsExtensions` を外す。
- TS 5.4+ なら `module: "preserve"` も選択肢。`verbatimModuleSyntax` との組み合わせがクリーン。

### §4.2 Node.js ESM application

`package.json` に `"type": "module"` 必須。相対 import は `.js` 拡張子を書く（ソースが `.ts` であっても）。

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
    "noUnusedParameters": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "dist"]
}
```

**TS 5.7+ なら `rewriteRelativeImportExtensions` を検討:**

```jsonc
{
  "compilerOptions": {
    "rewriteRelativeImportExtensions": true,
    "allowImportingTsExtensions": true
  }
}
```

これで `import { x } from "./foo.ts"` と書いて出力では `./foo.js` に書き換えてくれる。`.js` を手書きする煩雑さが消える。Node 22.6+ の `--experimental-strip-types` と組み合わせるなら `erasableSyntaxOnly` も併用。

レガシー CommonJS Node: `module: "commonjs"`, `moduleResolution: "node"`, `verbatimModuleSyntax` を外す、`package.json` から `"type": "module"` を消す。

### §4.3 Publishable library (separate JS and `.d.ts` output)

```json
{
  "compilerOptions": {
    "target": "es2022",
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

- `target` は consumer の Node 互換性で決める。Node 18 LTS で `es2022`、Node 20 LTS で `es2023`。`esnext` は consumer に不確実性を残すので避ける。
- 公開時の `package.json` `exports` は **各 conditional ブロック内で `types` を最初に**書く。`main`/`types` は top-level に fallback として残す。詳細パターン (subpath wildcards 含む) は full-reference.md §3。
- 非 tsc の `.d.ts` 生成ツール (e.g. `tsdown`, `oxc`) を使うなら `isolatedDeclarations: true` を検討。

### §4.4 Monorepo with project references

ルートの `tsconfig.json`:

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

各パッケージは共有 base を `extends` し、`composite` を立てる:

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

ビルドは `tsc --build`（または `tsc -b`）。ルートに対する素の `tsc` は references を解決しない。

### §4.5 Testing (補助 config)

`vitest` / `jest` / `bun:test` などのテストランナー用に、本体と分けて `tsconfig.test.json` を置く。

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": true,
    "types": ["vitest/globals", "node"]
  },
  "include": ["src/**/*", "tests/**/*", "**/*.test.ts", "**/*.spec.ts"]
}
```

ポイント: 本体側は `exclude` でテストを外し、テスト用 config は `include` で取り戻す。`types` でテストランナーの ambient 型を流し込むのが標準。IDE/editor が両方の config を認識するよう、エディタ拡張側で `tsconfig.test.json` を別 project として登録する場合もある。

---

## §5. Path aliases

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

Rules:

- `paths` は型検査だけに効く。bundler / runtime 側に同じ alias を別途設定する必要がある (Vite `resolve.alias`, webpack `resolve.alias`, Node import maps, etc.)。
- `moduleResolution: "bundler"` 環境では `baseUrl` は省略可。
- プロジェクトで alias スタイルは一つに統一。`@/` と `~/` を混ぜない。

---

## §6. Pitfalls — apply as rules, not theory

症状から原因と打ち手に直接落とす。

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Cannot find module './foo'` in Node ESM | import に `.js` が無い | `.js` 拡張子を付ける（`.ts` ソースでも）。TS 5.7+ なら `rewriteRelativeImportExtensions` で自動化。 |
| `paths` は tsc で動くが runtime で死ぬ | tsc は `noEmit` で import を書き換えない | bundler 側にも同じ alias を設定。 |
| `const enum` rejected | `isolatedModules: true` | `as const` オブジェクトに置換。 |
| Spread が `exactOptionalPropertyTypes` で壊れる | `{...x, y: maybeUndef}` で `y` が present-with-undefined になる | キーを条件付きで含めるか、`y?: T \| undefined` に広げる。 |
| Config 変更後も型エラーが残る | `.tsbuildinfo` のキャッシュ | `.tsbuildinfo` を消して再ビルド。 |
| `import type` 強制でビルドが壊れる | `verbatimModuleSyntax` + 値/型 mix import | 二つの import に分けるか `import { type X, y }` を使う。 |
| 壊れた `.d.ts` 一個でビルド全停止 | `skipLibCheck: false` | `skipLibCheck` を戻す。 |
| `exports` が型解決で間違ったファイルを返す | conditional 内で `types` が先頭でない | 各 conditional ブロックの先頭に `types` を移動。 |
| 副作用 import のタイポが検出されない | `noUncheckedSideEffectImports` が無い | TS 5.6+ なら有効化（§3）。 |

詳細・なぜ: full-reference.md §5 (pitfalls deep explanations)。

---

## §7. Diagnostic commands

config が「効いていない」「想定と違う」と感じたら、推測する前にこれらを走らせる。

```sh
tsc --showConfig            # extends 解決後の最終 config
tsc --traceResolution       # 全モジュール解決ステップ（巨大、ファイルに流すこと）
tsc --extendedDiagnostics   # フェーズ別タイミング（性能調査用）
tsc --listFiles             # コンパイル対象の全ファイル
```

`tsc --showConfig` が圧倒的に最重要かつ過小評価されている。`「動かない」と感じた瞬間に最初に走らせる`。

詳細 (incremental, tsBuildInfoFile 等の performance tuning) は full-reference.md §4。

---

## §8. Migration order (loose → strict)

既存プロジェクトを締めるときは、この順で有効化。各ステップ後に `tsc --noEmit` を走らせ、ステップ間でコミット。

1. `strict: true`
2. `noUnusedLocals` + `noUnusedParameters`
3. `noImplicitReturns` + `noFallthroughCasesInSwitch`
4. `noUncheckedIndexedAccess`
5. `exactOptionalPropertyTypes`
6. `verbatimModuleSyntax`
7. `noUncheckedSideEffectImports` (5.6+)

**全部一度に**有効化しない。エラー量が読めなくなり、バグ修正と churn が混ざる。

詳細 (PR 戦略、commits 粒度): full-reference.md §6。

---

## §9. Sharing config across packages

`tsconfig.base.json` をリポジトリルートに置き、各パッケージで `extends`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "./dist", "rootDir": "./src" },
  "include": ["src"]
}
```

TS 5.0+ なら `extends` は配列も取れる（レイヤード preset 用）:

```json
{ "extends": ["@tsconfig/strictest/tsconfig", "./tsconfig.base.json"] }
```

`@tsconfig/*` は TypeScript チームの公式リポジトリ (`github.com/tsconfig/bases`)。よく使われるもの:

- `@tsconfig/strictest` — 推奨 strict 全部入り。本スキルとほぼ一致する。
- `@tsconfig/node20`, `@tsconfig/node22` — runtime ターゲット別の defaults。
- `@tsconfig/recommended` — 最小 modern baseline。

---

## References

`references/full-reference.md` は SKILL.md より深い prose 解説を持つ:

| 知りたい | 読むべき節 |
| --- | --- |
| なぜこの policy か（哲学） | §1 |
| flag 個別の「なぜ」(strict, verbatimModuleSyntax 等) | §2 |
| `package.json` `exports` の詳細パターン (subpath wildcards 含む) | §3 |
| 大規模 monorepo の performance tuning | §4 |
| pitfall の prose 解説 (症状の裏側の仕組み) | §5 |
| migration の現場運用 (PR 粒度、commits 戦略) | §6 |

TypeScript 4.x の古い flag 名・挙動は本スキルではカバーしない。reference にも入れない（5.0 以降のみ）。
