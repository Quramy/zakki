# Relay v18 の `@throwOnFieldError` と GraphQL Nullability

## はじめに

Relay v18.0.0 にて `@throwOnFieldError` や `@catch`, `@semanticNonNull` といったエラーハンドリングに関連する Directive が追加された。
これらの Directive の意味と活用方法をこのエントリで解説していく。

なお、今回のエントリは自分が書いた以下エントリの続編としての意味合いも含んでいる。

https://quramy.medium.com/graphql-%E3%81%AE-semantic-non-null-type-rfc-%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6-49fb18a06afb

## Relay v18 で追加された Client Side Directive

### GraphQL Erorrs をコンポーネントから扱える

Relay v18 で追加された `@throwOnFieldError` および `@catch` はクライアントサイド, すなわち GraphQL オペレーション側に記述する Directive である。

以降の解説のため、次の GraphQL クエリを題材とする。

```gql
query {
  book {
    title
    author {
      name
    }
  }
}
```

このクエリに対して、`author` フィールドにてエラーが発生したしよう。この場合、レスポンスは `data` に正常に取得できた部分を、`errors` にエラーが発生した箇所の情報が格納される。

```json
{
  "data": {
    "book": {
      "title": "GraphQL Book",
      "author": null
    }
  },
  "errors": [{ "path": ["book", "author"], "message": "Something went wrong" }]
}
```

また、`data` 部におけるエラーが発生したフィールドの値は問答無用で `null` となる。これは GraphQL のエラーハンドリングを考える上で重要な性質であるため、後半で別途詳細を解説する。

上記の `errors` 部分は GraphQL の仕様に定められた挙動であるが、おそらく Relay ユーザーの場合、この GraphQL Errors を意識してきたことはあまりないのではなかろうか。というのは、Relay アプリケーションの場合、データアクセスは主に `useFragment` Hook 関数に頼ることになるが、クエリレスポンスにおける `errors` の情報が `useFragment` の結果には影響を及ぼさなかったからだ。

`@throwOnFieldError` や `@catch` を利用すると、クエリレスポンスで発生したエラー情報が `useFragment` 結果に伝達されるようになる。 Relay Runtime が `errors` に集約された情報を Leaf たる Fragment 側に分配してくれると言い換えることもできる。

以前に https://quramy.medium.com/graphql-error-%E4%B8%8B%E3%81%8B%E3%82%89%E8%A6%8B%E3%82%8B%E3%81%8B-%E6%A8%AA%E3%81%8B%E3%82%89%E8%A6%8B%E3%82%8B%E3%81%8B-3924880be51f にて、

> GraphQL の `errors` フィールドの場合、コロケーションとの組み合わせの相性があまりにも悪いのだ。

と書いたことがあるのだが、v18 で導入された Directive によって、この考え方も見直されたと言えよう。

エラー創出まで含めてコンポーネントに委ねることができるようになったというのは、コロケーション大好きな筆者にとって、好ましい機能追加である。

### `@throwOnFieldError`

まずは `@throwOnFieldError` から。この Directive は次のように Fragment 定義に付与する。

```jsx
function BookSummary({ fragmentRef }) {
  const data = useFragment(
    graphql`
      fragment BookSummary_Book on Book @throwOnFieldError {
        title
        author {
          name
        }
      }
    `,
    fragmentRef
  );

  // render using book data
}
```

上記の例では、`BookSummary_Book` Fragment のいずれかのフィールドに GraphQL Error が存在する場合に `useFragment` Hook それ自体が JavaScript Error を投げる。したがって `BookSummary` コンポーネントは描画されずに、上位の Error Boundary に補足されるまでエラーが React コンポーネントツリーの末端から頂点へ伝播していく。

### `@catch`

もう一つの Directive, `@catch` はフィールドに付与して利用する。`@throwOnFieldError` が GraphQL Error を JavaScript Error として throw するのに対し、`@catch` はその名の通り発生した GraphQL Error を捕捉するために用いる。

```jsx
function BookSummary({ fragmentRef }) {
  const data = useFragment(
    graphql`
      fragment BookSummary_Book on Book {
        title
        author @catch {
          name
        }
      }
    `,
    fragmentRef
  );

  if (!data.author.ok) {
    console.log(data.author.errors); // [{ message: "Something went wrong" }]
    return <AuthorFieldError />;
  }

  const author = data.author.value;

  // render using book and author data
}
```

上記の例は、`Book.author` フィールドそれ自身と配下のフィールドで GraphQL Error が発生した場合に反応する。コードでも例示した通り `@catch` Directive が付与されたフィールドは Result 型に変更されるため、正常系の値を取り出すには
`data.author.ok` が `true` であることを確認してから `data.author.value` のようにアクセスする必要がある。

```ts
type Result<T> =
  | {
      ok: true;
      value: T;
    }
  | {
      ok: false;
      errors: unknown[];
    };
```

`@catch` を多段階に適用させた場合、GraphQL Error が発生したフィールドから見て一番近い `@catch` に捕捉される。

```gql
fragment SomeFragment on AwesomeType {
  hoge @catch {
    fuga
    piyo @catch
  }
}
```

- `hoge` で GraphQL Error が発生した場合: `hoge` の `@catch` に捕捉される
- `hoge.fuga` で GraphQL Error が発生した場合: `hoge` の `@catch` に捕捉される
- `hoge.piyo`で GraphQL Error が発生した場合: `piyo` の `@catch` に捕捉される

## Semantic Non Null

ここからは `@semanticNonNull` Directive の解説をしていきたいのだが、その背景として GraphQL における Nullability について触れておく。

### GraphQL の Field Nullability

本エントリでは以下の GraphQL レスポンスを題材としてきた。

```json
{
  "data": {
    "book": {
      "title": "GraphQL Book",
      "author": null
    }
  },
  "errors": [{ "path": ["book", "author"], "message": "Something went wrong" }]
}
```

ところで、今回はレスポンスやクエリは例示してきたが、それらの源泉たる Schema は例示していない。
しかし、上記のレスポンスから Schema について分かることが１つだけある。 それは「`author` フィールドは絶対に Strict(= Non Null) Type ではない」ということだ。

```gql
type Book {
  author: User # User! であることはあり得ない
}
```

### Null Propagation とレジリエンス

仮に `Book.author` が Non Null Type であったとしよう。

```gql
type Book {
  title: String!
  author: User!
}

type User {
  name: String
}

type Query {
  book: Book
}
```

この場合 `Book.author` にてエラーが生じた場合、「`author` フィールドは Null ではない」という Schema で宣言した制約を満たせなくなる。このケースにおいては GraphQL の仕様上、より上位の Resolver 結果を Null にすることで、レスポンスと Schema 間の齟齬が生じないようにすることが定められており、 結果として `data.book` まで含めて Null となる。

ref: https://spec.graphql.org/October2021/#sec-Handling-Field-Errors

この仕組みは "Null Propagation" または "Null Bubbling" と呼ばれており、Schema 設計時の悩みの種となる。

前述の Schema について、書籍の情報を管理しているサービスと著者の情報を管理しているサービスが分散しており、GraphQL Resolver はそれぞれのマイクロサービスと通信していたとしよう。
`Book.author` が Non Null Type であるということは、何かしらの障害で著者情報管理サービスが応答不能になった場合に `Book.auhtor` だけでなく、参照可能である `Book.title` まで Null Bubbling に巻き込まれることを意味する。

このように、Non Null Type の利用は Schema としてのレジリエンスを低める方向に作用してしまうため、Type 間の参照は Nullable とすることが[ベストプラクティス](https://graphql.org/learn/best-practices/#nullability)とされている[^1]。

### Nullable Field が引き起こすトレードオフ

レジリエンス観点では Nullable Field でいいのだが、Schema を利用する立場からすると、「なんでここは Null になるんだ」と常々向き合う必要がでてくる。

業務的にも Null になり得るのであれば、クエリのデータが Null かどうかだけでなく「`errors` に当該パスを含むかどうか」まで見に行かないと、障害なのか、ただデータが存在しないだけなのか区別が行えない。

また、業務上 Null になり得ないという場合でも、Schema を見ただけではその意図は伝わらない。コメントで補足するか、または Union を使うことになる。

```gql
# コメントで補足するパターン
type Book {
  title: String!

  """
  Null になるのはエラー発生時だけ
  """
  author: User
}
```

```gql
# GraphQL Union で表現するパターン
type Book {
  title: String!
  author: AuthorResult!
}
union AuthorResult = User | ServiceError
```

また、フロントエンドは都度「エラーでないこと」を確認しないと `author` の中身にアクセスできないため、コードも冗長にならざるをえない。

```js
const data = useFragment(
  graphql`
    fragment BookSummary_Book on Book {
      author {
        name
      }
    }
  `,
  fragmentRef
);

if (!data.author) {
  throw new AssertionError();
}

console.log(data.author.name);
```

### `@semanticNonNull` Schema Directive

結局のところ、GraphQL Schema が「このフィールドは基本的には Null にはならない」を表明できていないことが混乱を生んでいる。

1. Non Null Type: field は確実に値をもつ: `author: User!`
2. Nullable Type: field は 業務上も Null となり得る: ???
3. Semantic Non Null Type: field は は障害が発生しない限り Null にはなり得ない: ???

GraphQL Schema が構造的に 2. と 3. の区別を付けられるようになっていれば以下のようなコメントは不要になるはずである。

```gql
type Book {
  title: String!

  """
  Null になるのはエラー発生時だけ
  """
  author: User
}
```

もちろん GraphQL として見分けを付けられるようにするためには、GraphQL Schema Definition Language(SDL) に何かしらの新しい文法を追加しなくてはならない。2024 年 10 月現在では [GraphQL Nullability WG](https://github.com/graphql/nullability-wg)で議論がなされている最中であり、どのような表現となるかは未定である(例えば、https://github.com/graphql/graphql-spec/pull/1065 には `!` を前置することで Semantic Non Null を表現する RFC である)。

そこで、暫定的に Semantic Non Null Type と Nullable Type を区別するために Relay の Jordan Eldredge 氏の主導のもとで考え出されたのが `@semanticNonNull` Schema Directive である。

```gql
directive @semanticNonNull(levels: [Int] = [0]) on FIELD_DEFINITION

type Book {
  title: String!
  author: User @semanticNonNull
}
```

上記のように、Semantic Non Null, すなわち「業務上は Null にはならない(がエラーが発生した場合を除く)」、の表明に用いる。

余談だが「これ RFC とすれば新しい Syntax は不要では？」と思う方がいるかもしれないので念の為に補足しておくと、GraphQL Schema Directive は飽くまで「その Schema の実装者に指示するため」のものであり、フロントエンドへの表明としては本来不十分である[^2]。

### `@semanticNonNull` と `@throwOnFieldError`, `@catch` の合せ技

前半に解説した `@throwOnFieldError` や `@catch` を使うことで、Relay コンポーネントはフィールドが Nullish かどうかだけでなく、そのフィールドがエラーだったかどうかまで認識できるようになった。また、Relay Compiler は当該フィールドに `@semanticNonNull` が付与されているかどうかを知っている。

Semantic Non Null Field の持つ「エラーが発生しない限りは Null ではない」と、`@throwOnFieldError` が持つ「フィールドでエラーが発生したらコンポーネントがエラーを投げる」を組み合わせると、コンポーネントから冗長な Null Check を排除できるのだ。

```gql
type Book {
  title: String!
  author: User @semanticNonNull
}
```

```tsx
function BookSummary({ fragmentRef }: Props) {
  const data = useFragment(
    graphql`
      fragment BookSummary_Book on Book @throwOnFieldError {
        title
        author {
          name
        }
      }
    `,
    fragmentRef
  );

  // data.author は TypeScript 上 Strict
  console.log(data.author.name);
}
```

上記は `@throwOnFieldError` の例であるが、 `@catch` においても、`data.author.ok` を確認したあとは、 `data.author.value` に Strict な型が手に入る[^3]。

なお、 `@semanticNonNull` は Relay 発の Schema Directive ではあるものの、Relay 以外のフレームワークでも対応しているものがある。

ref: https://www.apollographql.com/docs/kotlin/advanced/nullability#semanticnonnull

## v18 以降の Relay Null Handling スタンダード

ここまで Relay v18 で導入された 新規 Directive の利用方法とその存在意義を解説してきた。最後に、これらをどのように使うとよいかについて、現時点での筆者の考えを記載しておく。

### Nullable なフィールドへ `@semanticNonNull` を付与を検討する

まず、業務上は Null になり得ないのであれば、それはコメントではなく `@semanticNonNull` Directive を記載するように。

```gql
# Bad

type Book {
  title: String!

  """
  Null になるのはエラー発生時だけ
  """
  author: User
}
```

```gql
# Good

type Book {
  title: String!
  author: User @semanticNonNull
}
```

### `@throwOnFieldError` to Error Boundary

`useFragment` や `usePreloadedQuery` で用いる GraphQL Fragment, Operation の定義時には `@throwOnFieldError` を付与すること。これによりコンポーネントにおける冗長な Null Check が不要となる。

また、GraphQL errors がコンポーネントから throw されて、Error Boundary でキャッチされる、というのは React のコードとしても自然な形であると言えよう。
`useFragment` Hook 関数として見ると、Fragment が Deferrable な場合に Promise が throw されて、上位 コンポーネントの Suspense でキャッチするのと類似している[^4]。

Partial Error を考慮して細かく Error Boundary で Fragment Container から投げられたエラーをハンドリングしてもよいが、まずはアプリケーション最上位の Error Boundary でキャッチして「予期せぬエラーが発生しました」画面の表示に留めるもよしである。Partial Error については Schema Data Source がどこまで分散しているかなどのコンテキストによって、その必要性が異なってくるが、いずれにせよ通常の React アプリケーションのエラーハンドリング作法に乗っておくことがまずは重要。

### どこまで GraphQL Errors を許容するべきか

`@catch` や `@throwOnFieldError` によって、アプリケーションレイヤから GraphQL Errors を扱いやすくなったのは事実であるが、それでも所詮 `type GraphQLError = { message: string }` でしかない。
Resolver で Error Extension を使えばより複雑なオブジェクトを GraphQL Errors に詰め込めるが、Relay Compiler はその複雑な Extension を認識できるわけではない。
エラーハンドリングとして複雑な処理を行おうとしているのであれば、それは準正常系として Union なりで表現した方がよほど楽である。

```gql
type Book {
  title: String!
  author: AuthorResult!
}

type AuthorError {
  complexField: SomeComplexType!
}

union AuthorResult = User | AuthorError
```

これを考えると、コンポーネントそれ自身でエラーハンドリングを行う `@catch` よりは「異常系ハンドリングは別に任せる」の思想である `@throwOnFieldError` の方を優先して利用したい。

### `@required` は利用しない

Nullability の制御という意味では、Realy v17 で `@required` という Directive が追加されているが、これは最早使用しない方がよい。

ref: https://relay.dev/docs/guides/required-directive/

たとえば、以下は Resolver が `author` を Null として返却した場合、Relay が強制的に エラーを引き起こすことを引き換えにして `data.author` が Strict Type となる。以前、GraphQL Spec の RFC として CCN(Client Controlled Nullability) として提案されていた機能である。

```gql
fragment BookSummary_Book on Book {
  title
  author @required(action: THROW) {
    name
  }
}
```

`@required` は `@throwOnFieldError` とは違って GraphQL Errors を認識しているわけではない。クライアント側だけで「Null にはなりません！」と言っているイメージ。TypeScript の Non Null Assertion ととても似ているし、実際 `data.author!.title` としているのと挙動として大差がない。

もし、Semantic Non Null の意味で `@required` を利用しているのであれば、 `@semanticNonNull` と `@throwOnFieldError` で代替可能であるので、乗り換えを検討するとよい。

## Appendix. Resolver 実装と Semantic Non Null Field

本エントリはクライアントサイドたる Relay を中心に書いてきたが、サーバー側の実装についても捕捉しておく。

`@semanticNonNull` は「エラーにならない限り Null ではない」を表明するための Schema Directive であり、サーバー側はこの制約に準拠する必要がある。

```gql
type Book {
  title: String!
  author: User @semanticNonNull
}
```

すなわち、 `@semanticNonNull` が付与された Field Resolver の実装にて Null をエラーの代わりに用いるコードを書いてはならない。

```js
const Book = {
  author: async (parent, _, context) => {
    try {
      return await context.userServiceClient.fetchById(parent.authorId);
    } catch (err) {
      console.error("Unexpected error occurs.", err);

      // Bad
      return null;
    }
  },
};
```

以下のように修正すること。

```js
const Book = {
  author: async (parent, _, context) => {
    try {
      return await context.userServiceClient.fetchById(parent.authorId);
    } catch (err) {
      console.error("Unexpected error occurs.", err);

      // Good
      throw err;
    }
  },
};
```

筆者は TypeScript で GraphQL のサーバー側の実装を行うときは、[GraphQL-Codegen の typescript-resolvers プラグイン](https://the-guild.dev/graphql/codegen/plugins/typescript/typescript-resolvers) で TypeScript 用の Resolver Types を生成するのだが「 `@semanticNonNull` を付与したフィールドは Null を返却できない」というオプションあると、より安全に作業を進められるのではと考えている[^5]。

## 参考資料

- GraphQL Conf 2024 での Jordan Eldredge 氏による Semantic Nullability の解説動画: https://www.youtube.com/watch?v=kVYlplb1gKk
- GraphQL Nullability WG の概況説明: https://github.com/graphql/graphql.github.io/blob/nullability-post/src/pages/blog/2024-08-14-exploring-true-nullability.mdx

## 脚注

[^1]: レジリエンスだけでなく、Schema の後方互換性を保ちやすいという観点もある。
[^2]: Schema のメタデータ問題は太古の昔から未解決である。 https://github.com/graphql/graphql-spec/issues/300
[^3]: 厳密には relay-compiler の v18.0.0 では `@catch` 側は未実装. https://github.com/facebook/relay/pull/4794 で修正された
[^4]: `@defer` と Suspense については https://quramy.medium.com/render-as-you-fetch-incremental-graphql-fragments-70e643edd61e を参照されたし
[^5]: 少し前に自分で Issue と PR を書いた. 2024 年 10 月現在オープンなまま: Issue: https://github.com/dotansimha/graphql-code-generator/issues/10151, PR: https://github.com/dotansimha/graphql-code-generator/pull/10159
