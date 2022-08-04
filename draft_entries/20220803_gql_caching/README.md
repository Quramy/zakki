# GraphQL Client における Cache の話

## Client Caching

Relay や Apollo のスタンス:

- Performance 都合のみであれば、Document Cache でも十分。
- Cached Data の一貫性を保持することを考えると、Document Cache では不十分であり、正規化が必要

https://relay.dev/docs/principles-and-architecture/thinking-in-graphql/#client-caching

urql の場合、初期状態は Document Cache だが、ストアしたデータ間の依存管理が複雑となる場合には、正規化された Cache を opt-in で利用することを推奨している。

https://formidable.com/open-source/urql/docs/graphcache/

## State management と Normalize

Cache とは平たくいえば、SPA における Application State に相当と言い換えることができる。正規化については、GraphQL だけではなく、Redux の State を構造化する選リャとしてもよく用いられる。

https://redux.js.org/usage/structuring-reducers/normalizing-state-shape

GraphQL の場合, Output Type が型情報をもつため、Output Type ごとにテーブルを構成するイメージ。

余談だが、Relay は `Node` という Interface の実装を強制してくるのも、Client Side で Cache の一貫性を保つため。
`Node` Interface は まさにグラフデータ構造におけるノードであるが、Cache データ間で参照を保持するための一意識別子が必要となるからである。

```gql
interface Node {
  id: ID!
}

type Faction implements Node {
  id: ID!
  name: String
  ships: ShipConnection
}

type Ship implements Node {
  id: ID!
  name: String
}

type ShipConnection {
  edges: [ShipEdge]
  pageInfo: PageInfo!
}

type ShipEdge {
  cursor: String!
  node: Ship
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  rebels: Faction
  empire: Faction
  node(id: ID!): Node
}
```

## Apollo Cache

### Apollo Cache Overview

https://www.apollographql.com/docs/react/caching/overview/

HTTP で言う所の Cache-Control ヘッダに似た概念として、 `fetchPolicy` がある。

### Cache Policy

https://www.apollographql.com/docs/react/data/queries/#supported-fetch-policies

デフォルトは `cache-first` 、すなわち「キャッシュがあれば利用する」であり、 `Cache-Control: immutable` と同じ。

表示するデータを更新する経路が、そのアプリケーションを触っている人のみ ( `@current_user` のイメージ）であれば、上記で基本問題がない。

複数のユーザーが同時に編集するデータ構造が多い場合は以下あたりをデフォルトにしておくとよい。

- `network-only`: cache を信用しない。 `Cache-Control: no-cache` に近い
- `cache-and-network`: cache を利用するが、Server にも問い合わせる。 `Cache-Contro;: stale-while-revalidate` に近い

基本は `new ApolloClient` するときに、デフォルトの Fetch Policy を上記から選んでおき、必要に応じて、 `client.query` のオプションで override して使う。

### Cache key

Relay は `Node` interface Schema を schema に強制することで、正規化キャッシュの Primary Key が保証されるが、一般的な GraphQL Schema の場合、全ての Type に一意識別があるわけではない。

これが問題となるケースがある。

Apollo Client はオブジェクトのデータに `id` もしくは `_id` が含まれていることを暗黙的に仮定し、これを 正規化キャッシュの Primary Key として利用するようになっている。

```gql
type Query {
  messages: [Message!]!
}

type Message {
  seqNo: Int!
  text: String!
}
```

たとえば、上記の Schema は `Message` の識別子として `seqNo` を想定しているが、Apollo はこれを認識できない。

```gql
query {
  messages {
    seqNo
    text
  }
}
```

で取得しても、`Message::undefined` のような同一 ID として扱われて、画面が壊れて無事死亡する。

一番良いのはエンティティとして扱われる Type には全て `id` フィールドを用意しておくこと（特に Active Record であれば普通に model を書くと大体そうなる)。
どうしても難しければ、`typePolicies` や `dataIdFromObject` を適切に設定すること。

### Mutation と Cache

管理画面系などのアプリケーションで、model の CRUL がそのまま GraphQL Schema で表現されるような構造の場合、クライアントで正規化キャッシュの面倒を見ることが可能。 (Relay や Apollo は基本このスタンスを推奨している)。

この考え方に沿った場合、Mutation 実行後に Query 全体を再実行する必要がなくなるのが利点。
Schema 設計時は「Mutation は変更されるグラフ構造を部分的に返却する」を考えればよい。

(Demonstration)

ただし、アプリケーションの特性によっては対応が難しいケースも多い。
キャッシュにおける正規化は、RDB のそれと同じく、Single Source of Truth の思想が根底にあるが、GraphQL Schema のレベルでこれが成り立っていない場合はには正規化キャッシュの意味はどんどんぼやけることになる。

エンティティそのものではなく、Computed Value を field にもつような Type の割合が高いような Schema は、Mutation によるエンティティ更新が「どの Computed Field を変更するのか」を管理するのが難しくなっていくし、これを無理やり管理しようとしても恐らくキャッシュ更新漏れのような事故につながってしまう。

### Apollo Cache Future

おそらく、次の Major Version である Apollo Client v4 で大きく Cache と UI のインタラクションとなる API にも手が入りそう。

https://github.com/apollographql/apollo-client/issues/8245

特に v3 以降の Apollo Client は React 上で利用されることを前提とした Roadmap を敷いており、React v18 との親和性を考えた際に、ライブラリとしてのアーキテクチャを一部見直す必要が出てきてしまっているのが大きそう。

上記の issue を見た感じ、より Relay と似たようなライブラリになっていく雰囲気。

Apollo Client 側は、Relay と同じような `useFragment` API を誕生させようとしており、実際 v3.7 で一部利用可能になる見通し。

- https://github.com/apollographql/apollo-client/issues/8236

> useFragment is a read-only reactive/live binding into the cache, providing an always-up-to-date view of whatever data the cache currently contains for a given fragment.

Fragment を取り出す API が生まれる、ということは、Cache 層が蓄えているデータから、部分的なグラフ構造を任意に取り出すことができる、という意味になる。
(現状でも `InMemoryCache` に含まれる `readFragment` API で同じ様なことができなくはない)

## Document Cache / Normalized Cache と Incremental Delivery

パフォーマンスの観点では、Document Cache v.s. Normalized Cache に大きな違いは無いが、これは「一度の GraphQL Request が一回で全てのレスポンスを返却する」という世界限定の話。

GraphQL Spec では、 `@defer` や `@stream` といった Directive が検討されており、これらを有効にすると Query のレスポンスと Fragment のレスポンスが別々のものとして扱われるようになる。

すなわち、Store に溜まるタイミングも Query 本体とそこに含まれる Fragment で異なることになる。これらの機能が使えるように Apollo Client が発展していくことを考えると、Apollo Cache から Fragment（i.e. Document Cache ではなく、Normalized Cache から構成される部分グラフ）を取り出す機能が強化されていくのは違和感のない帰結といえる。

- https://quramy.medium.com/incremental-data-delivery-with-graphql-defer-and-stream-d779b5e38833
- https://quramy.medium.com/render-as-you-fetch-incremental-graphql-fragments-70e643edd61e
