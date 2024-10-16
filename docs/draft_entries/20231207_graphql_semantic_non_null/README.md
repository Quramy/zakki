# GraphQL の Semantic Non Null Type RFC について

これは [GraphQL Advent Calendar](https://qiita.com/advent-calendar/2023/graphql) ７日目の記事です。

_2024.10.16 追記_

本エントリのその後について、https://quramy.medium.com/relay-v18-%E3%81%AE-throwonfielderror-%E3%81%A8-graphql-nullability-55ce63c3aae2 に記載しているので、まずはこちらを読んでください。

_追記 ここまで_

GraphQL Specification に [RFC: SemanticNonNull type (null only on error)](https://github.com/graphql/graphql-spec/pull/1065) という RFC が上がっていたので、ざっと読んでみました。
2023.12 月現在では Stage 0 なので、取り込まれない可能性もおおいにあり得ることをご承知おきください。

この RFC は `[String]`, `String!` に加えて、`!String` のような型修飾のパターンを追加する、というものです。

`!String` のような、 `!` を前置した型修飾を "Semantic Non Null Type" と呼びます。
Semantic Non Null Type は「Data Response 上は null になり得るが、null になってよいのは、その path でエラーが生じたときだけ」を表します。

Semantic Non Null Type の必要性を考えるために、まずは従来からの (Strict な) Non Null Type のスキーマについて、エラー発生時に何が起こるかをおさらいしてみましょう。

```graphql
type Hoge {
  foo: String!
  bar: String
}

type Query {
  hoge: Hoge
}
```

`{ hoge { foo, bar } }` というクエリに対して、リゾルバが `Hoge.foo` の解決に失敗した場合、レスポンスは以下のようになります。

```json
{
  "data": {
    "hoge": null
  },
  "errors": [{ "path": ["hoge", "foo"], "message": "Unexpected error occurs" }]
}
```

`Hoge.foo` は Non Null Type であるため、このフィールドの値を null とすることはできません。
[GraphQL の仕様](https://spec.graphql.org/October2021/#sec-Handling-Field-Errors) では、Nullable な Type が出現するまで親 Type にエラーを伝播させることが定められています。

上記の例では 一階層上である `Query.hoge` が Nullable なフィールドであるため、`hoge: null` というレスポンスとなるわけです。もちろん、取得に成功したかもしれない `Hoge.bar` の情報をクライアントが参照することはできません。

この null の伝播を発生させたくないからといって、`Hoge.foo` の型を `String!` ではなく `String` とすると、クライアントは逐一フィールドの存在確認をしなくてはなりません。
もちろん正常系としてオプショナルなのであれば構いませんが、GraphQL Error 以外に null が発生する可能性が無いことが分かっている状況において、都度クライアントが値のチェックを強いられると、不要なコードが増えてしまいます。

「null の伝播を発生させたくない」と「不要な値チェックをしたくない」を両立させるのが、Semantic Non Null Type ということになります。「(正常系の範囲においては) 非 null」ということですね。

先程のスキーマの例に即すと、`Hoge.foo` が Semantic Non Null の場合、GraphQL Error が発生したとしても `data.hoge` が null になることもなく、`data.hoge.bar` の値の取り出しもできるわけです。

```json
{
  "data": {
    "hoge": {
      "foo": null,
      "bar": "BAR"
    }
  },
  "errors": [{ "path": ["hoge", "foo"], "message": "Unexpected error occurs" }]
}
```

GraphQL クライアントにとって、クエリからの型生成は必須と言ってよい存在ですが、Semantic Non Null なフィールドに対するクエリの型はどうすればよいのでしょう？
僕自身が https://github.com/Quramy/ts-graphql-plugin にて型生成器をメンテしている都合上、個人的にはクライアントの型やコード生成周りへの影響は気になるところです。

```graphql
fragment HogeFragment on Hoge {
  foo
}
```

上記の Fragment について、 `type HogeFragment = { foo: string }` のような「非 null なクライアントの型を生成してよいかどうか」は、GraphQL クライアントのランタイム次第で変わってくるはずです。

これについては [WG の議論](https://github.com/graphql/graphql-wg/discussions/1394) の方を参照するとわかりやすいです。

> A sufficiently smart client could parse the errors metadata of the response, and ensure that reading any GraphQL data that includes a field error results in an error.

レスポンスの `errors` 部分を使って GraphQL クライアントの側でエラーハンドリングをしてくれれば、`.foo` についての「(正常系の範囲においては) 非 null」性を保てる、ということですね。

なお、上記の GitHub Discussion を記述している `@captbaritone` さんは Relay チームの特に Relay Compiler 周りをやっている方なので、もし Semantic Non Null Type の RFC が取り込まれたのであれば、Relay Compiler は `!String` を Strict な型として出力するのだと思います。

> This is especially attractive for clients that encourage data colocation, where data is exposed to product code at a fragment granularity. This allows the blast radius of a field error to be limited to the fragment/component in which it was read.

また、Relay には [`@required`](https://relay.dev/docs/guides/required-directive) というディレクティブが存在しています。

> which means that any field you annotate with @required will become non-nullable in the generated types for your response.

とあるように、このディレクティブは「スキーマ上は Nullable な型であるフィールドに付与すると、そのフィールドに対応するクライアントの型は必須型となる」という性質のものです。
言い換えると、クライアント側にて Nullable なフィールドを Semantic Non Null とみなすのと同じです。

値を提供する側ではなく、利用する側が「これは必須だ」と言い張っているという意味では TypeScript の Non Null Assertion とも似ているかもしれませんね。

今回の RFC が実際に Spec に取り込まれるかどうかはまだわかりませんが、Semantic Non Null Type が利用できるようになれば、 `@required` を利用する必然性もなくなっていくのだと思います。

## 参考リンクなど

今年の GraphQL Conf 辺りから、Nullability の扱いが活発に議論されていました。今回の Semantic Non Null Type の RFC に収束するまでの経緯については、下記が参考になります。

- https://github.com/graphql/graphql-wg/blob/main/rfcs/ClientControlledNullability.md : Client Controlled Nullability(CCN) RFC
- https://graphql.org/conf/sessions/50005edb4a441b0335d1b80b4ad62b1a : GraphQL Conf 2023 での CCN の紹介動画
- https://github.com/graphql/graphql-wg/discussions/1410 : Lee Byron による Semantic Nullability の考察
