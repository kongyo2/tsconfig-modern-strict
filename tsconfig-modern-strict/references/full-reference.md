# TypeScript Configuration Reference (prose: the "why")

SKILL.md は「何を」「どう」並べるかの運用書。本ファイルは **その背後の「なぜ」** に絞った prose の解説。テンプレや表は SKILL.md にあり、ここでは繰り返さない。SKILL.md から特定の話題を深掘りしたいときに、対応する節を読みに来る形を想定する。

対象 TypeScript: 5.0 以降。それより古い版の挙動はカバーしない。

---

## 目次

- §1. Why ESNext + Maximum Strict（全体哲学）
- §2. Flag-by-flag rationale（各 flag の「なぜ」）
  - §2.1 `target` / `module` / `moduleResolution` のトリオが独立に決まらない理由
  - §2.2 `strict: true` が括る 8 フラグの個別解説
  - §2.3 `strict` に入らないが入れるべき追加フラグの個別解説
  - §2.4 `verbatimModuleSyntax` / `isolatedModules` がなぜペアで効くか
  - §2.5 `skipLibCheck` を切ってはいけない実務上の理由
- §3. Library publishing in depth（`package.json` `exports` の詳細パターン）
  - §3.1 conditional exports と `types` の評価順
  - §3.2 subpath wildcards
  - §3.3 dual-package (CJS + ESM) の落とし穴
  - §3.4 `isolatedDeclarations` を入れる判断
- §4. Performance tuning
  - §4.1 `incremental` / `composite` / `.tsbuildinfo` の仕組み
  - §4.2 Watch mode の高速化
  - §4.3 大規模 monorepo の references グラフ設計
- §5. Pitfalls deep explanations（症状の裏側の仕組み）
  - §5.1 Node ESM の拡張子要件はなぜ存在するか
  - §5.2 `exactOptionalPropertyTypes` で spread が壊れる本当の理由
  - §5.3 `paths` が runtime で死ぬ仕組み
  - §5.4 `isolatedModules` が `const enum` を拒む理由
  - §5.5 `.tsbuildinfo` 由来の幻のエラー
- §6. Migration strategy（現場運用）
  - §6.1 PR の粒度と commits 戦略
  - §6.2 大量エラーへの対処（@ts-expect-error の使い方）
  - §6.3 段階導入が破綻するサイン

---

## §1. Why ESNext + Maximum Strict

### tsc を「コードを書き換える人」と扱わず「型検査機」と扱う

2018-2020 頃の TypeScript 設定は、`target: "es5"` で polyfill を仕込み、`module: "commonjs"` で出力し、tsc 自身に runtime 互換性を担わせていた。これは bundler が貧弱で、Babel が型を知らず、Node が ESM をサポートしていなかった時代の事情に過ぎない。

2026 年現在、状況は反転している:

- **Bundler は target を別途持つ** (Vite の `build.target`, esbuild の `target`, webpack の `output.environment`)。tsc の `target` と二重で持つと、低い方が事実上の制約になり、tsc 側の `target` を絞っても利益はない（むしろ「現代の構文が使えない」と誤解を生む）。
- **Babel/SWC/esbuild は型を見ずに transpile する**。tsc に transpile を任せると、ビルド時間が数倍になる上に、何も得しない（型検査と transpile は別仕事）。
- **Node は ESM をネイティブサポート**。CommonJS で書く理由は legacy 互換だけ。

このため、本スキルは「**tsc は `noEmit` で型検査機としてだけ使い、target は ESNext に固定、emit は bundler / Node に任せる**」を原則に置く。library 公開のように tsc 自身が emit する場合だけ、consumer 互換のため `target` を引き下げる。

### なぜ「最大限」strict なのか

strict 系フラグは **後から有効化すると痛い**。`noUncheckedIndexedAccess` を未有効で 50,000 行書いたコードに後から入れると、何百箇所も `arr[i]!` か guard が必要になる。一方、最初から有効なら、コードを書きながら自然に `if (x !== undefined)` を挟むので、痛みは分散する。

「あとで strict 化する」は実務的に成立しにくい。strict は **新規プロジェクトの段階で固める** のが安い。既存プロジェクトの migration は §6 のように段階導入で抑える。

---

## §2. Flag-by-flag rationale

### §2.1 トリオが独立に決まらない理由

`target` / `module` / `moduleResolution` は、TypeScript の中で **互いに整合性を検査する**。たとえば:

- `module: "commonjs"` を選ぶと、`import` 文は `require()` に変換される。これに対して `moduleResolution: "bundler"` を組み合わせると、bundler 向けの解決（`package.json` の `exports` の `import` 条件を読む等）と CJS の出力が矛盾し、tsc がエラーを出す。
- `module: "nodenext"` は Node の ESM 仕様（`.js` 必須、`exports` 必須）を厳格にエミュレートする。これと `moduleResolution: "bundler"` を混ぜると、bundler が許す extensionless import を tsc が拒否する。

「アプリは `bundler`、Node ネイティブは `nodenext`」が現実的なルール。前者は bundler が解決方法を決め、後者は Node 自体が決める。中間（`module: "esnext"` + `moduleResolution: "node"`）は、レガシー Node の挙動を真似るだけで、`exports` 等の現代的な package metadata を読めず、デバッグが地獄になる。

### §2.2 `strict: true` が括る 8 フラグの個別解説

| Flag | 何を防ぐか・なぜ重要か |
| --- | --- |
| `noImplicitAny` | 暗黙の `any` は型検査の穴。`any` を入れたいなら明示する（`x: any`）。意図しない `any` は大半が「TS が型を推論できなかった = 何かおかしい」のシグナル。 |
| `strictNullChecks` | これを切ると `null`/`undefined` が任意の型に代入可能になり、型システムが事実上崩壊する。最重要フラグ。 |
| `strictFunctionTypes` | 関数引数の contravariance を強制。callback の引数を強い型で受けると、弱い型を渡せないようにする。React の event handler などで効く。 |
| `strictBindCallApply` | `fn.call(this, x, y)` の引数を `fn` のシグネチャと照合。`call`/`apply`/`bind` は型システムの抜け穴になりやすかったが、これで塞がれる。 |
| `strictPropertyInitialization` | クラスフィールドの初期化忘れを検出。`strictNullChecks` とペアで意味を持つ。`!` (definite assignment assertion) は最後の手段。 |
| `noImplicitThis` | `function() { this.x }` のような不明な `this` を拒否。arrow function や明示的な `this: Foo` パラメータを書かせる。 |
| `useUnknownInCatchVariables` | `catch (e)` の `e` を `unknown` 型に。`any` だと `e.message` が無条件で通り、想定外の throw を見逃す。`if (e instanceof Error)` で narrow する習慣を作る。 |
| `alwaysStrict` | `"use strict"` を全モジュールに付与し、parse 時も strict mode で扱う。`with` や `arguments.callee` などレガシー機能を禁止する。`esnext` を使う限りほぼ自動的に満たされるが、保険として残す。 |

### §2.3 `strict` 外の追加フラグの個別解説

| Flag | 何を防ぐか・なぜ重要か |
| --- | --- |
| `noUncheckedIndexedAccess` | `arr[i]` の型が `T` ではなく `T \| undefined` になる。配列の境界エラーは TypeScript で最も多いランタイム例外の一つで、これだけで激減する。代償は `arr[i]!` や guard を書く必要があること。`Map.get()` などの戻り型と一貫する形になる。 |
| `exactOptionalPropertyTypes` | `{ x?: number }` の `x` は「キーが無い」または「`number`」のどちらか。`{ x: undefined }` は許されない。これにより `JSON.stringify` の挙動（undefined は key ごと消える）と型が一致する。spread 系で痛い（§5.2）。 |
| `noImplicitOverride` | 親クラスのメソッドを上書きするとき `override` を必須に。親シグネチャをリネームしたとき、override のはずがオーバーロード扱いで silent に分岐する事故を防ぐ。 |
| `noImplicitReturns` | どこか一箇所で `return x` するなら、全パスで return する必要がある。`switch` の case で return 漏れがあるバグなどを潰す。 |
| `noFallthroughCasesInSwitch` | 空でない case が break/return せずに次の case に落ちる場合をエラーに。意図的な fallthrough は `// @ts-expect-error` ではなく `// falls through` コメントで明示する。 |
| `noUnusedLocals` / `noUnusedParameters` | 未使用変数・引数はエラー。引数だけは interface の都合で使えないことがあるので、`_` プレフィックスで個別 opt-out。グローバル無効化は「使わない引数を消す」という重要なリファクタを止めるので避ける。 |
| `noUncheckedSideEffectImports` (5.6+) | `import "./foo.js"` で `./foo.js` が存在しなくても TS は默って通していた（副作用 import と解釈）。これでタイポが検出される。 |

### §2.4 `verbatimModuleSyntax` / `isolatedModules` がなぜペアで効くか

esbuild、SWC、Babel は **ファイル単位** で transpile する。これらは型情報を持たないため、ある import が「型だけ」なのか「値も含む」のか判断できない。たとえば:

```ts
import { Foo } from "./foo";
```

`Foo` が型 alias なら、出力 JS では import 文ごと消すべき。クラスなら残すべき。tsc は型情報からこれを判断できるが、esbuild は判断できない。

`isolatedModules: true` は「ファイル単位で transpile 可能なコードしか書くな」という制約。`verbatimModuleSyntax: true` は「`import type` / `export type` を明示しろ、tsc は import 文を一切書き換えない」という意味。ペアで使うと、esbuild が見たままが tsc の意図と一致する。

`verbatimModuleSyntax` を切ると、tsc 側で型 import が値 import に化けて、esbuild 側の挙動と齟齬が出る。bundler のあるプロジェクトでは事実上必須。

### §2.5 `skipLibCheck` を切ってはいけない実務上の理由

`skipLibCheck: false` は依存パッケージの `.d.ts` を全て型検査する。問題は、依存ツリーに **壊れた `.d.ts` が一つでもあると、自プロジェクトのビルドが止まる** こと。これは:

- 依存パッケージのバグ（公開者がリリース時に型エラーを見逃した）
- 依存パッケージが古い TS で書かれ、新しい TS で型エラーになる
- 二つのパッケージが同じ型を別バージョンで定義し、merge で衝突

のいずれでも起きる。自分のコードの正しさには関係なく、ビルド可否が依存ツリーの状態に揺れる。利得は「依存パッケージの型エラーを検出できる」だが、自プロジェクトが影響を受けるなら issue を上げて修正を待つ方が安全で、ローカルで `skipLibCheck` を切る必要はない。

---

## §3. Library publishing in depth

ライブラリ公開では `tsconfig.json` だけでは不十分で、`package.json` 側の宣言が consumer の解決を決める。SKILL.md §4.3 のテンプレを書いた上で、`package.json` の `exports` と `files` を以下のように設計する。

### §3.1 conditional exports と `types` の評価順

```json
{
  "name": "@your-scope/package-name",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist-types/index.d.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "main": "./dist/index.js",
  "types": "./dist-types/index.d.ts",
  "files": ["dist", "dist-types", "!**/*.test.*", "!**/*.spec.*"]
}
```

**最重要ルール: 各 conditional ブロック内で `types` を一番上に置く**。Node / TypeScript / bundler の resolver は条件を上から順にマッチさせ、最初にマッチしたものを採用する。`types` が下にあると、`import` 条件が先にマッチして JS ファイルが返り、型解決が失敗する（または `.d.ts` を「見つからない」扱いになる）。

`"./package.json": "./package.json"` を必ず入れる。ツール（linters, bundlers）が `package.json` を読みに来るので、これが無いと `Module not found` になる。

top-level の `main` / `types` は exports をサポートしない古いツール（古い webpack、古い Node、古い editor）への fallback。残しておくのが安い。

### §3.2 subpath wildcards

複数エントリポイントを個別に列挙すると保守不能になる。`*` で一括宣言する:

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

`*` は consumer 側の subpath をキャプチャし、target 側の対応する `*` に展開する。`@your-scope/pkg/foo` → `./dist/foo.js`、`@your-scope/pkg/components/bar` → `./dist/components/bar.js`。target に対応するファイルが実在しないと、consumer のビルドで `Module not found` になるので、build 出力と一致するパターンを書く。

### §3.3 dual-package (CJS + ESM) の落とし穴

CJS と ESM を両方公開する `exports` を書くと、**同じモジュールが二つのインスタンスとしてロードされる**「dual-package hazard」が発生する。たとえば:

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

ある consumer が `require("@pkg")` し、別の consumer が `import "@pkg"` した場合、内部の `singleton` 変数が二つ存在することになる。これは `instanceof` の失敗、状態の二重保持などで実害を出す。

回避策:

- ESM-only で公開する（最も安全、依存元が CJS なら issue を上げる）
- `package.json` の `sideEffects: false` を設定し、純関数だけにする
- internal state を持つモジュールは必ず ESM のみで公開

新規パッケージは **ESM-only** が推奨。CJS consumer は `import()` で動的 import するか、ESM への移行を促す。

### §3.4 `isolatedDeclarations` を入れる判断

`isolatedDeclarations: true` (TS 5.5+) は、**ファイル単位で `.d.ts` が生成できるよう、public API に明示的な型注釈を要求する** フラグ。具体的には:

```ts
// NG: 戻り型を推論
export function foo() { return bar(); }

// OK
export function foo(): string { return bar(); }
```

これは tsc 以外の `.d.ts` 生成ツール（`tsdown`, `oxc`, `swc` の `dts` プラグイン等）が **whole-program inference 無しで** 型宣言ファイルを作れるようにするため。tsc 自身は whole-program で見るので不要だが、ビルドツールに JS 出力と型宣言を別系統で作らせる場合に意味がある。

判断:

- tsc だけで build しているなら **不要**（メリットなし、コストだけ）
- `tsdown` / `swc` で JS を、tsc で型を出す構成なら **入れて良い**（型ツールの選択肢が広がる）
- API 注釈の明示が読み手にも有益なので、純粋に design discipline として入れるのもアリ

---

## §4. Performance tuning

大規模プロジェクト（>100k LOC, monorepo）で tsc が遅いと感じたら、以下の順で対処する。

### §4.1 `incremental` / `composite` / `.tsbuildinfo` の仕組み

`incremental: true` を有効にすると、tsc は型検査結果のキャッシュを `.tsbuildinfo` ファイルに書き出す。次回実行時、変更されていないファイルの型検査をスキップする。デフォルトでは `outDir` の隣（`outDir` が無ければ `tsconfig.json` の隣）に置かれる。

`composite: true` は `incremental` を含意し、project references の前提条件でもある。composite な project は他の project から `references` で参照可能になる。

`tsBuildInfoFile` で `.tsbuildinfo` の場所を明示できる:

```json
{ "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo" }
```

これは CI で artifact をキャッシュする場合に便利。

注意: `compilerOptions` を変更したとき、tsc は `.tsbuildinfo` の互換性を完全には判定できない。変更後にエラーが残るときは `.tsbuildinfo` を削除して再実行する（§5.5 参照）。

### §4.2 Watch mode の高速化

`tsc --watch` の応答性を上げるには:

- `assumeChangesOnlyAffectDirectDependencies: true` で、変更ファイルの直接依存先だけを再検査する（推移的に追わない）。速いが、間接的な型壊しを見逃す可能性あり。
- `disableSourceOfProjectReferenceRedirect: true` で project references の解決を簡略化する（限定的なシナリオで有効）。

editor の TS server も同じ仕組みで動くので、これらは IDE 体感にも効く。

### §4.3 大規模 monorepo の references グラフ設計

`references` のグラフ設計の指針:

- **leaf package（依存されないが依存する側）** を増やしすぎない。一つのアプリパッケージから 20 個の lib を参照すると、`tsc --build` のスケジューリングが重くなる。
- **共通基盤を `tsconfig.base.json` で `extends`**、依存関係は `references` で表現するという二層構造を保つ。両者を混ぜると保守が破綻する。
- **CI では `tsc --build --verbose`** で実際のビルド順を確認し、想定外の依存があれば即修正する。

CI 観点では、`tsc --noEmit --incremental` を `.tsbuildinfo` を artifact キャッシュとして使うと、PR 間でキャッシュが効き、CI 時間が劇的に縮む。

---

## §5. Pitfalls deep explanations

SKILL.md §6 の症状表に対応する「裏側の仕組み」をここに残す。

### §5.1 Node ESM の拡張子要件はなぜ存在するか

Node ESM 仕様は **extensionless import を許さない**。これは「`./foo` が `./foo.js` か `./foo/index.js` か `./foo.ts` かを Node に判定させない」設計判断で、ES Module の仕様（仕様上拡張子の暗黙補完は無し）に従う。

TypeScript ソースファイルは `.ts` だが、Node の実行時は `.js` 拡張子で書く必要がある。これは「ソースは `.ts`、実行時の解決は `.js` を見る」という暗黙の規約。TS 5.7 の `--rewriteRelativeImportExtensions` は、ソースで `.ts` を書いて出力で `.js` に書き換えてくれる。これにより:

```ts
// Source (TS 5.7+ with rewriteRelativeImportExtensions)
import { x } from "./foo.ts";

// Output JS
import { x } from "./foo.js";
```

IDE での go-to-definition が `.ts` のまま動くので体験が良い。Node 22.6+ の `--experimental-strip-types` と組み合わせるとさらに強力（tsc が transpile せず、Node が type を剥ぐだけで動く）。

### §5.2 `exactOptionalPropertyTypes` で spread が壊れる本当の理由

```ts
type Opts = { x?: number };
const defaults: Opts = {};
const incoming: { x?: number | undefined } = { x: undefined };

const merged: Opts = { ...defaults, ...incoming };  // ERROR
```

`{ x?: number }` は **「`x` キーが無い」または「`number`」** という意味。`{ x: undefined }` は「`x` キーが存在し、値が `undefined`」なので、両者は別の型。

spread は object literal の代入なので、`incoming` の `x: undefined` がそのまま `merged` に入る → `merged.x` が present-with-undefined → `Opts` を満たさない、というロジック。

回避策:

1. キーを条件付きで含める: `const merged = { ...defaults, ...(incoming.x !== undefined ? { x: incoming.x } : {}) }`
2. 型を広げる: `type Opts = { x?: number | undefined }`（`undefined` 明示）
3. spread ではなく明示的なフィールド代入

3 が最も安全。spread は「型を見ずに rest が一致する」ことを期待する書き方で、`exactOptionalPropertyTypes` と本質的に相性が悪い。

### §5.3 `paths` が runtime で死ぬ仕組み

`paths` は tsc の型検査時のモジュール解決マップ。`tsc --emit` （古典的なビルド）では、tsc が出力 JS の `import` を書き換えて alias を解決する… **ことはない**。tsc は `paths` を「型検査だけのヒント」として扱い、emit 時には変換しない。

これは仕様。理由は「runtime の解決方法（bundler / Node / Deno）が tsc から独立しているべき」という設計判断。runtime に届くと `import { x } from "@/components/Foo"` のまま残り、Node や bundler が `@/components/Foo` を理解できずに `Module not found` になる。

正しい対処:

- Vite / webpack: `resolve.alias` に同じ alias を設定
- Node ESM: `package.json` の `imports` (subpath imports) や `--import` loader
- bundler を通さず tsc emit する場合: `tsc-alias` のような post-processor を使う（推奨しない、bundler を使うべき）

### §5.4 `isolatedModules` が `const enum` を拒む理由

`const enum` は、TypeScript が **コンパイル時に値をインライン化** する機能。

```ts
// Source
const enum Color { Red = 1, Blue = 2 }
const c = Color.Red;

// tsc emit (with const enum inlining)
const c = 1 /* Color.Red */;
```

これは tsc が **全プログラムを見て** 初めて可能。`isolatedModules` 環境（esbuild / SWC）はファイル単位で transpile するので、`Color` の定義を見ずに `Color.Red` を翻訳できない。

回避: 通常の `enum` を使うか、`as const` オブジェクトに置換する:

```ts
const Color = { Red: 1, Blue: 2 } as const;
type Color = (typeof Color)[keyof typeof Color];
```

後者が現代的。`enum` 自体も `erasableSyntaxOnly` (TS 5.8+) では禁止されるので、`as const` の方が将来安全。

### §5.5 `.tsbuildinfo` 由来の幻のエラー

`incremental` を有効にすると `.tsbuildinfo` がキャッシュを保持する。問題は、tsc は `compilerOptions` の変更が cached state に与える影響を **完全には追跡しない** こと。特に:

- `lib` の変更
- `paths` の追加
- `references` のグラフ変更

これらの後にエラーが「変な場所に残る」「変更したはずなのにキャッシュされた古いエラーが出る」と感じたら、`.tsbuildinfo` を削除して `tsc --build --force` で再生成する。

monorepo では各 package の `.tsbuildinfo` を全削除する必要がある場合がある:

```sh
find . -name "*.tsbuildinfo" -delete
```

---

## §6. Migration strategy

SKILL.md §8 の migration order（strict 段階導入）を、現場運用の観点で補足する。

### §6.1 PR の粒度と commits 戦略

各 strict flag の有効化は **独立した PR** にする。理由:

- 一つの flag を入れたときのエラー量が読める（10 か 1000 かで対応戦略が変わる）
- PR レビュアーが「何が変わったか」を理解できる（flag の意味と差分が一対一）
- 問題が出たら revert しやすい

PR 内の commits は:

1. `flag を有効化する commit`（tsconfig 変更のみ、テストは fail）
2. `errors を解消する commit`（または複数 commit、機械的修正と意味のある修正を分ける）

「機械的修正」と「意味のある修正」を混ぜない。前者は `_` プレフィックス追加、`!` non-null assertion、`as` cast など、レビュアーが流し読みできるもの。後者は実際にコードを変える修正で、設計判断が入る。

### §6.2 大量エラーへの対処

`noUncheckedIndexedAccess` を有効化すると、既存コードに数百のエラーが出ることがある。一気に直さず:

```ts
// migration 中の暫定対処
// @ts-expect-error TODO(strict-migration): noUncheckedIndexedAccess
const x = arr[i];
```

`@ts-expect-error` で **コメント付き** マークしておけば、後から `grep TODO(strict-migration)` で全箇所を集約できる。`@ts-ignore` は使わない（コメントが消えても気付かない）。`@ts-expect-error` は対象エラーが消えると逆にエラーになる。

各 PR で 50-100 件ずつ修正していくのが現実的。

### §6.3 段階導入が破綻するサイン

以下のいずれかが見えたら、migration を一時停止して全体を見直す:

- 一つの PR で 500 行以上の `!` / `as` が並ぶ → そもそもの API 設計が strict と合っていない可能性。型を見直す
- 同じ箇所を二回直している → migration 順序が悪い（後の flag が前の flag の修正を要求している）
- ランタイムバグが migration 期間に増えた → `!` で抑えた箇所が実際に undefined だった。`!` を guard に置き換える

段階導入は「型を整える」だけで「runtime を変える」べきではない。runtime バグが出るなら、migration の判定が間違っている。
