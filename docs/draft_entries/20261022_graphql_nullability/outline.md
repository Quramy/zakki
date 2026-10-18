# Title

GraphQL の `onError` と Nullability のこれから

# Description

- 対象: GraphQL を触ったことがある人
- GraphQL の Nullability にまつわる悩みと Semantic Non Null という考え方
- GraphQL Spec WG で議論されている `onError` リクエストパラメータの紹介

# Outlines

- 導入 (~0.5 min)
  - GraphQL Schema の `!`, どういう基準で付けていますか?
- Null Bubbling (~1.5 min)
  - 題材: `type Book { title: String! author: User! }`
  - `Book.author` の Resolver でエラーが発生した場合
    - `author` は `!` なので null にできない
    - 代わりに一番近い Nullable な親 (`book`) が null になる
    - 取得できていた `title` まで巻き添えで消える
  - だから「型間の参照は Nullable にしておけ」がベストプラクティス
    - ref: https://graphql.org/learn/best-practices/#nullability
- Nullable だらけの辛さ (~1 min)
  - 業務上は絶対に存在する `author` なのに, クライアントは毎回 null チェック
  - null を見ても「データがない」のか「エラーで取れなかった」のか区別できない
    - 区別するには `errors` の `path` を突き合わせる必要がある
  - 「Null になるのはエラーのときだけ」をコメントで書く Schema
- Semantic Non Null という考え方 (~2 min)
  - フィールドの性質は実は 3 種類
    - Semantically Nullable: 業務上 null がありうる. 例: `Book.coverImage`
    - Semantically Non-Null: null になるのはエラーのときだけ. 例: `Book.author`
    - Strictly Non-Null: エラーにも null にもならない. 例: `Book.id`
  - 今の GraphQL で書けるのは `Int` と `Int!` の 2 種類だけ. 真ん中が書けない
  - これまでの試み
    - `@semanticNonNull` Directive (Relay, Apollo Kotlin が対応)
    - 新しい型を足す案 (`author: User*` など) は Spec WG で議論されたが失速
- `onError` とは (~2.5 min)
  - 発想の転換: 新しい型を足すのではなく, Null Bubbling の方を止められるようにする
  - Request の属性. Query の構文ではない
    - `query`, `variables`, `operationName` と並ぶ
    - HTTP なら JSON body に `"onError": "NULL"`
  - 値は 3 つ: `PROPAGATE` / `NULL` / `HALT`
  - 同じ Schema, 同じ Query, `Book.author` でエラーが発生した場合のレスポンス比較
    - `PROPAGATE`: `book` ごと null (従来通り)
    - `NULL`: `book.title` は残り, `author` だけ null
    - `HALT`: レスポンス全体がエラー
- `onError` で何が変わるか (~1.5 min)
  - `NULL` なら `!` の位置が null になるのは「エラーが起きたときだけ」
    - つまり `!` がそのまま Semantic Non Null の意味になる
    - Schema 設計者は「業務上 null になりうるか」だけ考えて `!` を付ければよい
  - 引き換えに, エラーを扱う責務はクライアントへ
    - null を読む前に `errors` を見て throw する
    - Relay: `@throwOnFieldError`, それ以外: graphql-toe
    - React なら Error Boundary で受ける
- 現状 (~1 min)
  - まだ RFC 1 (提案段階). 2026 年版の Spec には入らない見込み
  - デフォルトは今後も `PROPAGATE`. 既存の Query の意味は変わらない
  - 実装: Apollo Kotlin, Hot Chocolate はマージ済. graphql-js, graphql-java は PR 中
- まとめ (~0.5 min)
  - `onError` で, エラー伝播の有無をリクエストごとに選べるようになる
  - その結果, `!` を「業務上 null にならない」の意味で付けられるようになる
    - Semantic Non Null を表す新しい型を Schema に足す必要はなくなる, というのが今の WG の方向性
    - ただし `PROPAGATE` のクライアントが残る間は `@semanticNonNull` も移行用として残る

# Appendix 候補 (質疑用)

- CCN (Client Controlled Nullability) との違い
  - CCN は Query 側でフィールドごとに `!` / `?` を付ける案. 2023-12 に Rejected
  - 「クライアントが Schema の不正確さを補う」から「Schema が正しく表現する」へ方針転換
- サーバーが `onError` に対応しているかの検出: Service Capabilities (#1208)
- 前回のブログ: ../20241016_error_handling_in_relay_18

# References

- https://github.com/graphql/graphql-spec/pull/1163
- https://github.com/graphql/graphql-spec/pull/1236
- https://github.com/graphql/graphql-wg/blob/main/rfcs/SemanticNullability.md
- https://github.com/graphql/graphql-wg/blob/main/notes/2026/summary-2026-08-06.md
- https://benjie.dev/graphql/nullability
