TypeScript 4.1 の RC がリリースされましたね。

https://devblogs.microsoft.com/typescript/announcing-typescript-4-1-rc/

## New Features

### Template Literal Types

JS の Template String Literal の如く、String Literal Type のなかで、さらに別の type を placeholder として使えるようになる機能です。

```ts
type Hello<T extends string> = `Hello ${S}`;
type HelloWorld = Hello<'world'>; // 'Hello world' という Literal Type
```

Hejlsberg が PR 作った時点で、（一部界隈では）話題になりましたね。
Conditional Types の推論と組み合わせれば、字句解析器(Lexer)が作れてしまうので、型レベルでオレオレ言語が作れてしまうことに。

https://github.com/microsoft/TypeScript/pull/40336 のコメント欄が最早大喜利の様相を呈しています。

#### Key Remapping in Mapped Types

Mapped Types の key を `{ [P in T as NewType<P>]: ValueType<T, P> }` のように `as ...` で変換することができる機能。
前術の Template Literal Types と組み合わせると、型レベルとしての snake case と camel case の変換など、色々なことができるようになります。

この機能と合わせて、組み込みの Utility Types として、 `Capitalize`, `Uncapitalize`, `Uppercase`, `Lowercase` という 4 つが追加されています。

```ts
type X = Uppercase<"hoge">; // 'HOGE' type
```

みたいな感じですね。

Template Literal Type と Key Remapping in Mapped Types は、新しい構文の追加となるため、ESLint や Prettier を使っている場合、 `@typescript-eslint/estree` が 4.1 対応されるまでは利用を控えた方が良いです。

https://github.com/typescript-eslint/typescript-eslint/issues/2583

### Recursive Conditional Types

読んで字の如く、Conditional Types で再帰が許容されるようになりました。

4.0 までの TS では下記のようなに自己を参照する Conditional Types を記述するとエラーとなってしまいました（このせいで遅延評価を利用した hack が横行していた）。
Template Literal Types + Conditional Types を有効に活用するためには、とても重要な機能です。

```ts
type Awaited<T> = T extends PromiseLike<infer U> ? Awaited<U> : T;
```

この制限が 4.1 で緩和されます。ただし、型計算の複雑度チェックは依然として有効なため、あまり複雑すぎる再帰型は割とすぐにエラーになります。

また、この制限緩和の影響を受けて、3.8 で議論された `awaited` keyword は merge されることなく廃止となります。

### --noUncheckedIndexedAccess

`arr[1]` のような index access をする際に、その値の存在確認を強制するためのオプションです。 `--strict` を付与しただけでは有効化されないので、Prj を始めるときは自分で有効化するようにすると良いでしょう。

```ts
const arr: string[] = [];
console.log(arr[1].toUpperCase()); // arr[1]の存在を確認する前にアクセスするとエラー
```

### --jsx="react-jsx"

React.js v17 の https://reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html の transpile 制御を TypeScript で行うためのオプションです。

従来の `--jsx="react"` の場合、 `jsxFactory` （通常は `React` ） が tsx ファイル中で declare されていることが強制されるため、通常は下記の import 文を行頭に書いていました。

```ts
import React from "react";
```

こうすることで、 TypeScript は jsx 部分を `React.createElement` に transform していました。

`--jsx="react-jsx"` とすると、 TypeScript の transformer が自動で

```ts
import { jsx as _jsx } from "react/jsx-runtime";
```

を挿入し、且つ `_jsx` を `React.createElement` の代わりに利用するように挙動が変わります。なお、import 先を変更したい場合は、 `@jsxImportSource` という pragma が提供されているので、これを使って変更することができます（tsconfig からは変更できません）。

また、「transpile 自体を Babel にまかせていて、TypeScript は type check でしか利用していない」というケースであっても、 `--jsx="react"` だと 「React という Identifier がそのモジュールで定義されていること」のチェックが走ってしまうため、

```json
[
  "@babel/preset-react",
  {
    "runtime": "automatic"
  }
]
```

の設定をしている場合は、 `--jsx="react-jsx"` に変更した方が良いでしょう。

### Editor Support for the JSDoc @see Tag

JSDoc コメント内で `@see <ジャンプ先のIdentifier>` と書くと、Go To Definition でジャンプできるようになりました。
地味ですが便利な機能なので、積極的に使っていきたいですね。

### その他

blog には記載がありませんが、 `--generateTraces` というオプションが増えてます。Type Checker や Transformer の詳細な profile を取得するためのオプションです。
取得した結果は V8 の CPU profile のように、Chrome の devtool から確認できます。
自分のプロジェクトで tsc があまりにも遅いと思ったら、計測してみると良いかもしれません。

## Breaking Changes

割と踏みそうなやつを 2 点だけ紹介。

### any / unknown と`&&` 演算子

```ts
declare let foo: unknown;
declare let somethingElse: { someProp: string };

let x = foo && somethingElse;
```

今までは、 `x` は `{ someProp: string }` に推論されてしまったのですが、 4.1 からは `unknown` に推論されるようになります。
User Type Guard Function で、 `return obj && typeof obj.x === 'string'` のような書き方をしていたりすると、エラーになりますね。

### Promise における resolve のパラメータが必須に

```ts
new Promsie(resolve => {
  resolve();
});
```

上記のように、引数なしでの resolve 呼び出しがエラー扱いされます。

4.0 までは下記のように書いてしまったとしてもエラーにならなかったのですが、これが修正されたためです。

```ts
new Promsie<number>(resolve => {
  resolve(); // 本来number型の引数を与えるべき
});
```

`resolve()` したい場合は、引数が不要であることを Generics で明示するようにしましょう。

```ts
new Promsie<void>(resolve => {
  resolve();
});
```
