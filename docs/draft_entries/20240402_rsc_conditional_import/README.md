# Server Component と Client Component で依存モジュールを切り替える

ちょっとした React Server Component 小ネタ。

Next.js (webpack, Turbopack) で確認しているが、おそらく RSC に対応しているツールであったらどれも変わらないはず。

アプリケーションの package.json の `imports` セクションに以下のように記載しておく。`util` の部分は好きな文字列で構わないが `#` から始めておくこと。

```json
{
  "imports": {
    "#util": {
      "react-server": "./src/util.react-server.ts",
      "default": "./src/util.default.ts"
    }
  }
}
```

React Server 環境とそれ以外の環境用、それぞれの実装を用意する。
とりあえず結果が異なることを確認したければ以下のような感じ。

```ts
/* ./src/util.default.ts */

export function awesomeFn() {
  return "Default env";
}
```

```ts
/* ./src/util.react-server.ts */

export function awesomeFn() {
  return "React Server env";
}
```

基本的に準備はこれで終わり。あとはアプリケーションのコンポーネントで `"#util"` からインポートするように書けば、そのコンポーネントが SC(Server Component) か CC(Client Component) かによって、解決されるモジュール実態が切り替わる。

```ts
import { awesomeFn } from "#util";

// このコンポーネントが SC か CC かによって出力結果が切り替わる
function MyComponent() {
  return <div>{awesomeFn()}</div>;
}
```

## 仕組

冒頭に記載した package.json の `imports` の部分は Node.js の Subpath imports と呼ばれる機能。

https://nodejs.org/api/packages.html#subpath-imports

Node.js 自体が気にする Condition 名は `import` や `require` であるが、Community Conditions として任意の値を利用することができる (たとえば、 TypeScript は `import`, `require`, `default` に加えて、`types` というキーを識別する)。

React は React Server 環境を表す目的で `react-server` を使ようになっている。React 本体もそうであるが、`server-only` や `client-only` パッケージの利用例が一番わかりやすい。
`server-only` の package.json は以下のようになっていて、React Server 環境以外だと index.js が読み込まれ、その瞬間にエラーとなる仕掛けになっている。

```json
{
  "name": "server-only",
  "files": ["index.js", "empty.js"],
  "main": "index.js",
  "exports": {
    ".": {
      "react-server": "./empty.js",
      "default": "./index.js"
    }
  }
}
```

```js
/* index.js */

throw new Error(
  "This module cannot be imported from a Client Component module. " +
    "It should only be used from a Server Component."
);
```

https://www.npmjs.com/package/server-only

一方で、React や 上記の `server-only` を使うためには、メタフレームワーク側が「これは React Server 環境で実行していますよ」を伝える必要があり、例えば Next.js は webpack の `resolve.conditionNames` を用いている。Turbopack の実装はちゃんと確認していないが、おそらく似たような設定項目があるのだと思う（手元で `next --turbo` で確認したらちゃんと動いたし)。

https://github.com/vercel/next.js/blob/v14.2.0-canary.51/packages/next/src/build/webpack-config.ts#L1351

## 使い所

紹介はしてみたものの、アプリケーションサイドがカジュアルに使い倒すものでは無いと考えている。

似たような機構に React Native の Platform Specific Extensions (`.ios.js` とか `.android.js` などで実行環境ごとのモジュールを切り替えるヤツ) がある。
iOS / Android の違いは横並びの概念であるが、RSC 環境かどうかというのは横並びというよりはむしろ事前・事後の関係に近い感覚なので、実装の異なるモジュールを import したいという要件が発生する頻度は React Native と比べると低いのではないかと思う。
実際、App Router の実案件をやっていてこのテクニックが必要だった箇所というのも思い当たらない。

また、TypeScript は Subpath imports について `defualt` や `types` までは認識するが、`react-server` を認識するわけではない。`use server` な Component から `import "#util"` にコードジャンプしても CC 向けである `default` の実装を開くため、デフォルト側の実装にコメントなりを記載しておかないと、Conditional Imports を利用していることが伝わらないだろうし。

<!--
以前に App Router な Next.js の案件でフィーチャートグルを環境変数で実現したことがあったのだけど、以下のような実装をしていた。

- SC: `process.env.ENABLE_FEATURE` 値を直接返却する
- CC: Client Boundary な Provider で `process.env.ENABLE_FEATURE` を Context に格納しておき、`useContext` で参照する

「フィーチャートグル値を参照する」という要件に対して、呼び出し元が SC か CC かで利用する関数を切り替えるような実装になっていて、少し初見者殺しなところがあったのだけど、`import { getFeatureToggleValue } from "#util"` のようにすれば
以前に App Router な Next.js の案件でフィーチャートグルを環境変数で実現したことがあったのだけど、以下のような実装をしていた。

- SC: `process.env.ENABLE_FEATURE` 値を直接返却する
- CC: Client Boundary な Provider で `process.env.ENABLE_FEATURE` を Context に格納しておき、`useContext` で参照する

「フィーチャートグル値を参照する」という要件に対して、呼び出し元が SC か CC かで利用する関数を切り替えるような実装になっていて、少し初見者殺しなところがあったのだけど、`import { getFeatureToggleValue } from "#util"` のようにすれば
-->
