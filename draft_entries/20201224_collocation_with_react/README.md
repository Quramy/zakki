# React での GraphQL Collocation 再考

これは [GraphQL Advent Calendar 2020](https://qiita.com/advent-calendar/2020/graphql) の 24 日目の記事です。

## はじめに

前回の defer/stream の記事は途中で力尽きてしまったため、クライアント側の処理についてはあまり多くを書けなかったので、この投稿で続きを書いていこうと思う。
アドカレの担当を途中で 2 つに増やしたから記事を分けようとかそういうセコいあれではないよ！

クライアント側の処理、それも React 限定の話だ。 `@defer` / `@stream` の「サーバーからちょっとずつデータが届けられる」という特性と、React の Suspense for data fetch の関係を考えてみたい。

なお、 `@defer` や `@stream` と毎回ディレクティブ名で言及するのが面倒なので、こちらについては Incremental data delivery と呼ぶことにする。

## Collocation

Incremental data delivery や Suspense for data fetch の話をする前に、くどいようだけど GraphQL のコロケーションについて触れておきたい。

僕は GraphQL の コロケーションという考え方が本当に好きで「GraphQL でクライアントサイドのアプリケーションを書くのであれば絶対に採用すべき」と考えている。コロケーションが何なのか、という話は他のところでも何度か書いているんだけど、今回はこの投稿の主要なテーマであるのであらためて触れておく。

GraphQL が RESTish な API と本質的に異なっているのは、GraphQL は Demand driven なデータの取得方法であるという点だ。Demand、すなわち「データを必要とする側がレスポンスの型を宣言する」ということだ。この点だけをいみると、むしろ SQL に近い。 SQL と違うのは `SELECT * FROM HOGE` のような、「全部まるっとよこせ」というやり方が構文のレベルで許容されていない点。

よく「GraphQL はクライアントが自由にレスポンスの形式を指定できて便利」と言われる。確かにそのとおりだと思うが、世の中に責任の伴わない自由など存在しない。
ここでいう責任とは「クエリを必要十分に保つこと」だ。必要なフィールドを書き忘れたら、それは機能障害だ。不必要なフィールドを書いたら over-fetching となり、サーバーリソースや帯域の無駄遣いになる。
GraphQL を採用する言うことは、フロントエンドがこれらに対する責任を負うということだ。

では GraphQL のクエリが、その画面にとって必要十分であり続けるためにはどうすればいいだろう。画面が複数のコンポーネントの階層から構成される場合、どのフィールドが必要になるかを知っているのは、ページのルートではなく、末端である個々のコンポーネントだ。それぞれのコンポーネントがクエリの断片として必要になるフィールドを管理しておき、それらを子から親に集約させながらルートとなるコンポーネントで一つのクエリを組み立てるようにすればいい。これが GraphQL におけるコロケーションの基本的な考え方だ。

TODO 図

`FragmentA` がどのようなセレクションを行うかは、A というコンポーネントが責任を持つべき事柄であるし、A のコンポーネントにさらに A-a がネストされている場合、A-a に必要なフィールドは `FragmentA-a` に管理を任せればよい。

.graphql ファイルでもいいし、 JSX ファイルの中に Template String として書いたっていい。それは些末な問題だ。

## Collocation と Incremental data delivery

さて、ここまでは良いだろうか。これを読んでいる貴方が GraphQL と React を使ってアプリケーションを作っているけどコロケーションはしていない、ということであれば、こんな記事を読んでいる場合ではないので、今すぐにこの画面を閉じて Fragment を切り出す作業に取り掛かって欲しい。ここから先の話はその作業が完了してから読めばいい（別に読まなくたっていい）。

ここからが本題だ。

前回の投稿で書いた通り、 `@defer` や `@stream` というディレクティブの登場によって、GraphQL は「ちょっとずつクエリの結果をクライアントに送信する」という手段を得たことになる。

おさらいも兼ねるが、 `@defer` はフラグメントに付与することで、そのフラグメントのレスポンスの取得を後回しにするものだ。 `@stream` はリストの要素に付与する。

`@defer` を使ったクエリの例は下記のようになる。

```graphql
fragment ProductPriceFragment on Product {
  specialPrice # 計算がちょっと重たい
}

query ProductDetailQuery {
  product(id: 100) {
    id
    name
    ...ProductPriceFragment @defer
  }
}
```

`@defer` は `@skip` などとは違い、フィールドには直接付与できないため、ここでは `ProductPriceFragment` というフラグメントを用意している。

ここで丁度フラグメントが出てきたので、コロケーションと組み合わせて実装してみたとしよう。

Apollo Client を使っているのであれば下記のような感じになるだろうか。ちなみに Apollo Client は `@defer` が実装されているわけではないため、下のコードを書いても動かないので、飽くまでイメージだ。

```js
export ProductPriceFragment = gql`
  fragment ProductPriceFragment on Product {
    specialPrice
  }
`;

const ProductPrice = ({ product }) => (
  <div>Special Price: {product.specialPrice}</div>
);

export default ProductPrice;
```

上記の `ProductPrice` というコンポーネントがフラグメントを export しており、それをクエリを使う側のコンポーネントで利用する。

```js
import ProductPrice, { ProductPriceFragment } from "./ProductPrice";

const query = gql`
  ${ProductPriceFragment}

  query ProductDetailQuery {
    product(id: 100) {
      id
      name
      ...ProductPriceFragment @defer
    }
  }
`;

const ProductDetail = () => {
  const { data, isLoading } = useQuery(query);
  if (isLoading) return <div>loading...</div>;
  return (
    <div>
      <span>{data.product.name}</span>
      <ProductPrice product={data.product} />
    </div>
  );
};
```

サーバーが `@defer` に対応していれば、このクエリは次のように分割された応答が返却されるはずだ。

```js
// レスポンス その1
{
  "data": {
    "product": {
      "id": 100,
      "name": "とても良い商品"
    }
  },
  "hasNext": true
}

// レスポンス その2
{
  "path": ["product"],
  "data": {
    "specialPrice": 1000
  },
  "hasNext": false
}
```

そう、ここで問題が出てくる。

## 現象の再整理

1 つのフィールドなのに一々フラグメントが必要な仕様になっている時点で恣意的なものを感じた人は
