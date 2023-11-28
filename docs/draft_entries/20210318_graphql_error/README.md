# GraphQL Error、下から見るか？横から見るか？

タイトルに深い意味はなくて、思いついちゃったので書いてみただけです。

GraphQL API を設計するときに「Query の一部がエラーとなってしまった場合に、そのエラー情報をどう伝播させるか」という部分について、悩みの種になりそうだなーっていっつも思ってたのだけど、実はもう決着がついていそう。

結論を先に書いてしまうと、「Partial Error については GraphQL errors field を使わずに、Schema 上で model として設計しろ」をベストプラクティスとして良さそう。

Facebook Relay がアプリケーションエラーについて下記のように見解を示していたことを社内で教えてもらったのがきっかけ。

https://relay.dev/docs/guided-tour/rendering/error-states/#accessing-errors-in-graphql-responses

> If you wish to access error information in your application to display user friendly messages, the recommended approach is to model and expose the error information as part of your GraphQL schema.

要するに、ユーザーにわかりやすいエラー情報を表示したければ、Schema の Model 情報として error を設計しろよ、ということだ。

そのまま Relay のドキュメントから転載するが、下記のように明示的に Schema 上で Error type を表す。

```graphql
type Error {
  # User friendly message
  message: String!
}

type Foo {
  bar: Result | Error
}
```

クエリ側からは、下のように Type Condition を使ってエラーか正常にデータ取得ができたかを判別する。

```graphql
fragment fooFragment {
  bar {
    __typename
    ... on Result {
      # 正常系のセレクション
    }
    ... on Error {
      message
    }
  }
}
```

TypeScript であれば、次のような型を上記の fragment から生成できていれば、「エラーハンドリングして型を確定しないと正常系の property にアクセスできない」という制約を課すのも簡単だ。

```ts
export type FooFragment = {
  readonly bar:
    | {
        __typename: "Result";
        // 正常系に相当するproperty
      }
    | {
        __typename: "Error";
        message: string;
      };
};
```

一方、GraphQL 自体に備わっている `errors` から エラー情報（どのフィールドが取得できなかったのか）を拾う、という選択肢もあるにはある。こちらは GraphQL が spec として用意した方法。

```js
const queryResultIncludingErrors = {
  errors: [
    {
      message: "なんかだめ",
      path: ["hoge", "foo", "bar"]
    }
  ],
  data: {
    hoge: {
      foo: {
        // 正常に取得できた部分
      }
    }
  }
};
```

冒頭の Relay のドキュメントを読んだ際に「なんで Relay は（GraphQL に用意されている方法をつかわずに）エラーを Schema で表現する方法を推しているんだろう？」という疑問が湧いた（how だけが書いてあって why が見当たらなかった）。

GraphQL のエラー設計自体についての考察としてはそれほど新しい話でもなくて、上記のパターンの比較も色々と書いてあるんだけど、一番納得感があったのが https://blog.logrocket.com/handling-graphql-errors-like-a-champ-with-unions-and-interfaces/ 。

> The errors are not collocated to where they occur

の一文に集約されていると感じた。

GraphQL の `errors` フィールドの場合、コロケーションと組み合わせとの相性があまりにも悪いのだ。

コロケーションを活用している場合、Fragment とともに対となる Component が存在しているはずで、その Component からしたら「末端の情報にしか興味がない」状態であり、逆に集約したクエリを叩く層から見ると、Fragment で管理されている枝葉末節についてのエラーを伝えられても困る」となる。

`useFragment` や `FragmentContainer` のように、Fragment を一級市民として扱ってきた Relay においては、エラー情報も Fragment に詰まっている方が圧倒的に扱いやすいわけだ。と勝手に納得感を得た。

ところで、GraphQL 自体も元はと言えば Facebook が自社のために作っていたプロトコルなわけで、仕様側はエラー設計についてどう考えてるんだろと思い、ついでに調べてみたところ、Lee Byron も下記のように書いていた。

> Since discussing this issue, a common best practice has been to include user errors as part of the schema itself so they can contain domain specific information.

https://github.com/graphql/graphql-spec/issues/135#issuecomment-426164615
