# CCN, `@semanticNonNull`, `onError` の比較

## 概要

### CCN (Client Controlled Nullability)

- Query 側でフィールドごとに Nullability を上書きする構文
  - `author!`: Non-Null として扱う. null だったらエラーにして伝播させる
  - `author?`: Nullable として扱う. エラー伝播の境界になる
- 2021-10 提案 (graphql-spec#895), 2023-12 Rejected
- 「クライアントが Schema の不正確さを補う」から「Schema が正しく Nullability を表現する」へ方針転換したため棄却
- Relay の `@required` はクライアントだけで完結する類似機能

```graphql
query {
  book {
    title
    author! {
      name
    }
  }
}
```

### `@semanticNonNull`

- 「null になるのはエラーのときだけ」を表す Schema Directive
- Relay 発 (Jordan Eldredge 氏主導). Apollo Kotlin なども対応
- GraphQL Spec 外. サーバーの実行には影響しないメタ情報で, 解釈するのはクライアントのツール
- `@throwOnFieldError` などと組み合わせると, クライアントは Non-Null 型として扱える

```graphql
type Book {
  title: String!
  author: User @semanticNonNull
}
```

### `onError`

- Request の属性. `query`, `variables`, `operationName` と並ぶ
- 値は `PROPAGATE` (従来通り, デフォルト) / `NULL` (伝播しない) / `HALT` (エラーが起きたら全体をエラーに)
- 2026-10 時点で RFC 1 (graphql-spec#1163, #1236)
- `NULL` なら `!` の位置が null になるのはエラーのときだけ. つまり `!` がそのまま Semantic Non-Null の意味になる

```json
{
  "query": "query { book { title author { name } } }",
  "onError": "NULL"
}
```

## 比較

|                              | CCN                          | `@semanticNonNull`               | `onError`                      |
| ---------------------------- | ---------------------------- | -------------------------------- | ------------------------------ |
| 種類                         | Query の構文                 | Schema Directive                 | Request の属性                 |
| 書く人                       | クライアント                 | Schema 設計者                    | クライアント                   |
| 指定の単位                   | フィールド                   | フィールド (リストは階層ごと)    | リクエスト全体                 |
| Nullability を決めるのは     | クライアント                 | Schema                           | Schema (`!`)                   |
| サーバーの実行への影響       | あり (伝播の境界が変わる)    | なし                             | あり (伝播の on/off)           |
| エラーの影響範囲を決めるのは | Query の `!` / `?` の位置    | クライアントの Error Boundary    | クライアントの Error Boundary  |
| Spec 上の状況                | Rejected (2023-12)           | Spec 外                          | RFC 1                          |

## 関係

1. CCN: クライアントが Nullability を決める案. 棄却
2. True Schema Nullability: Schema が「業務上 null になりうるか」を正しく表現する方針へ
3. Semantic Non-Null: そのために「null はエラーのときだけ」を表す手段が必要
   - `@semanticNonNull` Directive で先行実装
   - 新しい型を Spec に足す案 (`User*` など) は失速
4. `onError: NULL`: 伝播を止めれば `!` がそのまま Semantic Non-Null の意味になる. 新しい型は不要
   - ただし `PROPAGATE` のクライアントが残る間, `@semanticNonNull` は移行用として意味が残る
