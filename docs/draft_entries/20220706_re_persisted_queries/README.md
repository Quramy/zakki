# Persisted Query についての補足

## Persisted Query の副次的な効果

### CDC, Request Spec

GraphQL の世界では、Schema の範囲の中で許されるクエリのパターンは無限に存在する。

バックエンドからすると「クライアントからリクエストされうるクエリのパターンが高々有限個に限られている」となれば、その限られたパターンを Request Spec として Integration test を記述する、というような活用方法も考えられる。

これはクライアント(= Consumer) のクエリによって、API の決め事が合意されるという意味において、Consumer Driven Contract のアナロジーと言える。

参考:

- https://docs.pact.io/
- https://speakerdeck.com/qsona/graphql-for-service-to-service-communication-protocol?slide=22

### 情報の可視化文脈

例えば、Fragment Colocation を活用しているフロントエンドの場合、クエリ / フラグメントはレポジトリに散在することになる。

```gql
# ソースコー上は UserInfo Componentに記載されている
fragment UserInfo_User on User {
  id
  name
  avatarURL
}

# ソースコー上は Dashboard Page Component に記載されている
query DashboardPage_query {
  ...UserInfo_User
}
```

Persisted Query がワークフローに組み込まれているケースでは、フラグメントが統合されたクエリの形式で、フロントエンド - バックエンド間でやり取りされるため、バックエンド側からもクエリを認識しやすくなる。 また、適切なネーミングルールがあれば「クエリを元にどの画面で利用されているのか」の情報も手に入りやすい。

### バンドルサイズへの寄与

フロントエンドは、クエリ本文そのものではなく、そのハッシュさえ知っていればリクエストを送信できることになるため、バンドルサイズの削減に寄与する。

クエリそのものの長さ、というのもあるが、それだけではなく Runtime に持ち出される `graphql/language` (parse, print 関数など)を削減できる可能性がある。

※ 実際の導入難度は利用しているコード生成ツールに依存。

## Caveat

導入のコストがそれなりに必要。

1. バックエンドが Schema を作成する
1. フロントエンドが Schema を元にクエリと UI コンポーネントを作成する
1. フロントエンドが Persisted Query を生成する
1. バックエンドが生成された Persisted Query の受け入れ確認と登録を行う

また、登録した Persisted Query を削除するタイミングについても運用ルールを策定する必要がある（そうしないと際限なく増えていくことになってしまう）。

## Persisted Query を抽出するためのツール

- Apollo CLI: https://www.apollographql.com/docs/devtools/cli
  - `apollo client:extract` というコマンドで、クエリのハッシュ & 本文がペアになった JSON ファイルが出力される
- Relay Compiler: https://relay.dev/docs/guides/persisted-queries/#local-persisted-queries
  - `relay-compiler` 実行時に、同様の構造のファイルが手に入る
