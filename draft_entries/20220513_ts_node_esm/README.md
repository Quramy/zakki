# TypeScript 4.7 と Native Node.js ESM

TypeScript 4.7 の RC がリリースされたので、Node.js ESM 対応の現状をまとめておく。

@teppeis さんの [TypeScript 4.5 以降で ESM 対応はどうなるのか？](https://zenn.dev/teppeis/articles/2021-10-typescript-45-esm) を先に読んでおくと良い。

このエントリの中でも、teppeis さんの定義した用語をそのまま用いさせてもらう。

> - CommonJS (CJS): 従来式の Node.js CommonJS で書かれたファイルまたはパッケージ
> - ES Modules (ESM): ES2015 で定義されたモジュール仕様。Node.js では v12 以降でネイティブにサポートされている。
> - Native ESM: ESM 形式で記述されたファイルを、Node.js またはブラウザで直接 ESM として実行する方式またはそのファイル。擬似 ESM と区別するために Native と付けて呼ぶ。
> - 擬似 ESM: ESM 形式で記述されたファイルを、実行前に TypeScript や Babel で CJS に変換する方式またはそのファイル。ランタイムでは、Node.js は変換後のファイルを CJS として実行する。faux-ESM に対する私の訳語。
> - Pure ESM: Native ESM だけで構成する npm パッケージ。CJS からは require() で読むことはできず Dynamic Import import() を使って非同期に読む必要がある。

## 総論

TypeScript としての Node.js Native ESM サポートはシンプルで、拡張子が増えたことと、ESM / CJS ごとに型定義を読み分けられるようになったことくらい。
逆に言うと、機能の大半を理解するのには Node.js の ESM 対応（mjs や cjs、package.json の `exports` プロパティなど）を押さえておく必要がある。

一方で、TS + npm なパッケージ開発者がすぐに自身のパッケージを Native ESM 化できるかというと、懸念がありそう。依存パッケージの型定義周りが障壁になる可能性がある。
また、周辺ツールなども盤石とは言い難い状況であることから、無理してまですぐに対応するほどの温度感ではないな、という温度感。

## TS 4.5 と TS 4.7 の差分

TypeScript の Native ESM サポートについては、[4.5 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-4-5-beta/#esm-nodejs) の頃から色々情報が出てきているので、その主だった差分を書いておく。

- TypeScript 4.5 の時点では、Node.js ESM サポートは Nightly Build でしか利用できない実験的な機能であったが、4.7 で晴れて安定版の位置付けに
- 4.5 の時点では `--module node12` という名前だったが、4.7 以降では `--module node16` に変更されている。挙動自体は `node12` と同じ。Top Level Await についての解釈を考慮したときに、Node.js v12 よりも「Node.js v16 に対応している」と考えた方が綺麗だったため。
- Type Reference Directive 内で利用可能な `module-resolution` オプションが追加されている
- `--moduleDetection` という Compiler Option が追加された。スクリプトかそうじゃないかの挙動の判定に関係するが、基本的に気にする必要はない。

他にも Language Service 関連などのの細かい変更は諸々入っているはず。

## Node.js における Native ESM 対応

先に Node.js の Native ESM に対するアプローチをおさらいしておく。

[Node.js determines module type without file content](https://nodejs.org/api/packages.html#determining-module-system) にある通り、対象のファイルを ESM として読み込むのか、CJS として読み込むのかを**拡張子だけ**で決定する。

| JavaScript source file extension | Module Type                                                               |
| :------------------------------- | :------------------------------------------------------------------------ |
| `.js`                            | package.json に `"type": "module"` の記述があれば ESM, そうでなければ CJS |
| `.mjs`                           | 常に ESM                                                                  |
| `.cjs`                           | 常に CJS                                                                  |

なお、ESM と CJS を混在して扱う場合、以下のルールに従う。

| import(require) するファイル | import(require) されるファイル | Static Import | Dynamic Import | require |
| :--------------------------- | :----------------------------- | :------------ | :------------- | :------ |
| ESM                          | ESM                            | OK            | OK             | NG      |
| CJS                          | CJS                            | NG            | NG             | OK      |
| ESM                          | CJS                            | OK            | NG             | NG      |
| CJS                          | ESM                            | NG            | OK             | NG      |

また、CommonJS や 疑似 ESM で利用可能であった、 `__dirname` や `process` などは Native ESM からはアクセスできない。

## TypeScript における Node.js の Native ESM サポート

### 拡張子

Node.js が拡張子を使い分けたことに合わせて、TypeScript にも対応する新しい拡張子が導入された。

| TypeScript source file extension | Compiled JavaScript file extension | Generated type declaration file extension |
| :------------------------------- | :--------------------------------- | :---------------------------------------- |
| `.ts`                            | `.js`                              | `.d.ts`                                   |
| `.cts`                           | `.cjs`                             | `.d.cts`                                  |
| `.mts`                           | `.mjs`                             | `.d.mts`                                  |

新しく `.mts` と `.cts` が追加されたが、非常にシンプルで、tsc ではそれぞれ `.mjs` と `.cjs` にコンパイルされるだけ。

### `--module: node16` がやってくれること

上述の拡張子の対応を念頭に置いた場合、 `.ts` ファイルの最終的な Node.js での解釈は package.json に依存することになる。 `--module node16` では、

- package.json が `"type": "module"` であれば、.ts を Native ESM と解釈して、Import / Export Statement をそのまま出力する(従来における `--module esnext`)
- package.json が `"type": "module"` でなければ、.ts を CommonJS と解釈して、Import / Export Statement は `require` / `module.exports=...` に変換する(従来における `--module commonjs`)

もちろん、 `.mts` であれば package.json と関係なく前者となるし、`.cts` であれば後者。

なお、 `--module commonjs` を指定した場合は、ソースコードのファイルの拡張子や package.json の `"type"` フィールドと関係なく、 Import / Export Statement は `reuiqre` / `module.exports=...` に変換されるし、 `--module exnext` であれば、.cts の Import / Export Statement は保存される。

### Module Specifier

`import { Hoge } from "hoge"` に `"hoge"` のような「読み込みたいモジュールがどこにあるのか」の部分を Module Specifier と呼ぶ。

Node.js Native ESM の世界では、以下のような拡張子省略は許容されない。これは、ESM のルールではないが、Node.js ESM Loader が [WHATWG Loader Spec の Browser Loader](https://whatwg.github.io/loader/#properties-of-the-browser-loader-prototype) に追従しているため。

```js
import "./hoge";
```

拡張子省略が許容されないのは、 TypeScript の `--module node16` や `--module nodenext` の世界においても同様で、相対ファイルから Import する場合には、Module Specifier に拡張子を明記しなくてはならない。

```ts
import "./hoge.mjs"; // ./hoge.mts ではない！
```

ここで重要なのは、Module Specifier に記述するのは TypeScript の拡張子ではなく **「JavaScript にトランスパイルされた後の世界における拡張子」** であること。

「`"./hoge"` や `"./hoge.mts"` のように書きたい」という気持ちも一定理解できるが、これは TypeScript が[「Module Sepecifer を書き換えない」というポリシーを選択](https://github.com/microsoft/TypeScript/wiki/TypeScript-Design-Goals/53ffa9b1802cd8e18dfe4b2cd4e9ef5d4182df10#goals)しているため。

TypeScript コントロールしてくれるのは、「Module Specifer に対応する TypeScript ソースコードや型定義ファイルがどこにあるのかを解決する」ことだけ。
これを制御しているのが tsconfig における `moduleResolution` プロパティ。 実際 `--module node16` とすると裏では `--moduleResolution node16` にセットされる。

### Conditional Exports と 型定義ファイルの出し分け

`--module esnext` の世界でも、Pure ESM な Node.js パッケージを構成することはできた（ [sindresorhus](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c)が推しているヤツ）。

`--module node16` の世界では、ESM / CJS ファイルを混在させることができるようになり、拡張子も Node.js のルールに合わせてくれるようになった、というのがここまでの話。

TypeScript 4.7 では、`.mts`, `.cts` ファイル(または `type: module` でないパッケージでの `.ts` )を外部に公開する際に、付随する型定義ファイルも出し分けることができるようになっている。

これも Node.js の [Conditional exports feature](https://nodejs.org/api/packages.html#conditional-exports) を拡張する形になっている。

`--module node16` の場合、package.json に以下のように書かれていれば、適用させる Type Declaration を Node.js の Entry Point の判別に即して切り替えることができる。

```json
{
  "name": "@types/hoge",
  "main": "lib/index.js",
  "types": "lib/index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./lib_esm/index.d.mts",
        "default": "./lib_esm/index.mjs"
      },
      "require": {
        "types": "./lib_cjs/index.d.cts",
        "default": "./lib_cjs/index.cjs"
      }
    }
  }
}
```

`"require"` や `"import"` の各ブロックにも `"types"` フィールドを書いているところがポイント。

上記のように記載することによって、ESM としてパッケージを利用するユーザーと CJS としてパッケージを利用するユーザーに別々の型定義を提供することができる。

型定義の出し分けについて、恩恵が一番わかりやすいのが `@types/node` だと思うのだけど、2022 年 5 月現在 Definitely Typed の issue / PRs を軽く検索した感じ、関連するものがなさそう。

```ts
import fs from "node:fs";

// このファイルがCommonJSとして扱われるのであればOK, ESMとして扱われるのであればerrorにしたい
console.log(__dirname);
```

### `resolution-mode` による明示的な読み分け

逆に hoge パッケージを利用する側から明示的に読み分けることもできる。`resolution-mode` オプションが Triple Slash Directive Comment でかけるようになった。

```ts
/// <reference types="hoge" resolution-mode="require" />

/// <reference types="hoge" resolution-mode="import" />
```

正直、これらのユースケースを理解できていない。 Tripe Slash Directive なので、Definitely Typed の Contributor 向け？

`resolution-mode` については Import Assertion っぽい記法も導入された。ただし、この記法については、4.7 RC で最後の最後に Nightly でしか使えないようになっている。

```ts
import type { Hoge } from "hoge" assert { "resolution-mode": "require" };

import type { Hoge } from "hoge" assert { "resolution-mode": "import" };
```

https://github.com/microsoft/TypeScript/issues/48644 Nightly に格下げされた経緯が書かれているが、そもそもこの Syntax が意味的に正しいのかが微妙、といった理由があるようで、十分なフィードバックが得られるまでは正式版としたくない模様。

### `--module node16` と footgun

上述した「Type declaration の出し分けが可能になる」に付随して「利用しているパッケージが正しく Type declaration を出分けていないと、利用する側で不都合が生じうる」という話。

具体例を考えてみる。とある npm パッケージ `hoge` が以下のように構成されていたとする。

```json
{
  "name": "hoge",
  "main": "index.js",
  "types": "index.d.ts",
  "exports": {
    ".": {
      "import": "./lib_esm/index.mjs",
      "require": "./lib_cjs/index.cjs"
    }
  }
}
```

このパッケージを使っている側が、ts 4.6 までであれば、以下のコードは問題なくコンパイルできる。

```ts
/* main.ts */
import * as hoge from "hoge";
```

しかし、上記のファイルを `module: "node16"` として扱おうとした場合に、次の問題が起きる

- Node.js の世界では `./lib_esm/index.mjs` は利用可能なのでランタイム上は問題ない
- TypeScript 4.7 の世界においては `./lib_esm/index.d.mts` のファイルが存在しなければ、 `./lib_esm/index.mjs` の型定義が解決できずにエラーになる

このシナリオについては https://github.com/microsoft/TypeScript/issues/46334 で議論されており、方向性として「トップレベルの `types` を exports map の各ブロックにマージするようなことはしない」となっている。

これは、Node.js における Conditional Export が「明示的にエントリポイントを指定する機能」であるため、その考え方に準じてのこと。

したがって、このシナリオに遭遇した場合、hoge パッケージ(依存対象側)の側で、のエントリポイントごとの型定義の場所も明示するように修正する必要が出てくる。

```json
{
  "name": "hoge",
  "main": "index.js",
  "types": "index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./index.d.ts",
        "default": "./lib_esm/index.mjs"
      },
      "require": {
        "types": "./index.d.ts",
        "default": "./lib_cjs/index.cjs"
      }
    }
  }
}
```

この状況にぶち当たった場合、hoge パッケージの提供元に 修正 PR 送って取り込んでもらわねばならなくなる。もしかすると Ambient Module 宣言と前述の Import Type Statement + `"resultion-mode"` でどうにかできるかもしれないが、贔屓目で見たとしても推奨された方法ではないはず。。。

逆に、**自身で Conditional Exported な Package を公開するのであれば、それぞれのエントリポイントについて `"types"` フィールドの明記と同梱を忘れてはいけない**。

### ESM - CJS Interop

こちらもパッケージ利用者泣かせになりうる件。

似非 ESM を Native ESM から import した場合に、特に Default Export 周りで問題が発生する可能性がある。

```ts
/* source ./lib.ts */
const x = "value";
export default x;
```

```js
/* dist ./lib.js */
const x = "value";
exports.default = x;
```

```ts
/* dist ./lib.d.ts */
declare const x = "value";
export default x;
```

これらが `lib` となっていたとして、これを Native ESM から import すると、以下のコードのコンパイルが通らなくてはならない。

```js
/* ./index.mts */

import lib from "lib";

console.log(lib.default); // <- default が必要
```

[TypeScript blog](https://devblogs.microsoft.com/typescript/announcing-typescript-4-7-rc/#commonjs-interop) には、「厳密なのは無理だけど、ヒューリスティックに頑張るよ」的なことが書いてあるが、正直この条件が不明。

> There isn’t always a way for TypeScript to know whether these named imports will be synthesized, but TypeScript will err on being permissive and use some heuristics when importing from a file that is definitely a CommonJS module.

Conditional Import に従って、import 対象のモジュールの型定義ファイルの resolution-mode 相当がわかれば、CJS interop の必要有無がわかる、ということだろうか...?

## エコシステム

### `@types/node`

前述の通り、型定義の読み分けに対応している気配がないため、現状では「.mts で `__filename` などを使わないようにする」を自分で気をつけるしかない。

### Jest

Jest は Native ESM では利用できることはできるものの、[Node.js の `vm.Module`](https://nodejs.org/api/vm.html#class-vmmodule) を利用するようになっており、これは 起動時に `--experimental-vm-modules` が必要であることからも分かる通り、実験的な代物である状態なので、利用は自己責任で。

https://jestjs.io/docs/28.0/ecmascript-modules

TypeScript の文脈で注意すべきことは特に無いように思う。 .ts / .cts / .mts が混在しているプロジェクトであっても、結局は「拡張子に応じてモジュールの種別が決まる」というだけであるので、拡張子ごとに Transpiler の設定を適切に行えば問題ない。

以下に swc と 本家 TypeScript を 使った場合の Jest 設定ファイルの例をリンクしておく（Quramy 自身は普段は ts-jest を使っているのだけど、今回は Peer Dependencies 周りが面倒だったので諦めた）。

- `@swc/jest`
  - [Jest 設定](https://github.com/Quramy/ts-node-conditional-export-example/blob/39b8656337ff593308ec64f5a8d515ed634900a3/packages/library-pkg/jest.config.mjs)
- 自前で `ts.TranspileModule`
  - [Jest 設定](https://github.com/Quramy/ts-node-conditional-export-example/blob/89d53d21469ab41bcaa05eebb310d48c5e6f36eb/packages/library-pkg/jest.config.mjs)
  - [Transformer 本体](https://github.com/Quramy/ts-node-conditional-export-example/blob/89d53d21469ab41bcaa05eebb310d48c5e6f36eb/packages/jest/src/index.ts)

### ts-node

https://github.com/TypeStrong/ts-node/issues/1007 で議論されてはいるが、2022 年 5 月対応で未対応.

Description や [コメント](https://github.com/TypeStrong/ts-node/issues/1007#issuecomment-712967491) を読むと、Jest のような Experimental な Node.js の機能に頼るつもりはなさそうに見える。

[対応用の PR らしきもの](https://github.com/TypeStrong/ts-node/pull/1694) は見かけたものの、しばらくコミットがないので怪しい。
