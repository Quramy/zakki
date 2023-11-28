# Integrated testing with Prisma

このエントリは [Recruit Engineers Advent Calendar 2022](https://adventar.org/calendars/7972)の 4 日目の記事です。

![](https://images.unsplash.com/photo-1597600159211-d6c104f408d1?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=640&q=80)

Photo by [Lukas Tennie](https://unsplash.com/ja/@lazycreekimages) from unsplash

## はじめに

おしごとにて [Prisma ORM](https://www.prisma.io/) を使った Node.js + TypeScript なバックエンドサービスを開発・運用しています。

Prisma を利用する上で、テストの書きづらさがあったため、OSS を 2 つほど作って改善しました。今回のエントリでは、この 2 つの npm パッケージを中心に、Prisma のテスト周りについて書いていきます。

- https://github.com/Quramy/jest-prisma : 実 DB を使ったテストを書きやすくするためのツール
- https://github.com/Quramy/prisma-fabbrica : テストデータのセットアップを書きやすくする

どちらについても、無いからと言って Prisma を利用したアプリケーションのロジックがテストできないわけではありません。一方で、これは Storycap や reg-suit などの VRT 関連のツールを整備していた頃も強く感じたことなのですが、テストをきちんと書く文化を根付かせるためには「テストコード書くのダルい」とメンバーに感じさせない工夫は重要と思っています。

## 実 DB を使ったテストを書きやすくする

### 背景

自動テストにおいて、Prisma に実 DB と通信させること自体は難しくありません。

[Prisma のドキュメント](https://www.prisma.io/docs/guides/testing/integration-testing) にも Docker Compose で PostgreSQL を起動しつつ、Jest で テストスイートを動作させる方法についての記載があります。また CI に同様の環境を用意するのも難しくはありません。例えば GitHub Actions で Integrated な自動テストを行いたいのであれば、[Service Container](https://docs.github.com/en/actions/using-containerized-services/about-service-containers#example-mapping-redis-ports) を使えば簡単に実現できます。

ただし、テスト用のクリーンなデータベースがあるとはいえ、以下の課題が残ります。

- テストケースの実行毎に DB の状態を初期状態に戻す必要がある。Truncate All + seeding 相当を実行しなくてはならない
- 他のテストスイートで DB の状態が変更される可能性があるため、並行にテストを実行できない。 Jest であれば `--maxWorkers=1` が必須

これは Prisma / Node.js に限った話ではなく実 DB を利用してテストを実行する場合に総じて課題になる話です。したがって JavaScript 以外の言語では、どのようにこの課題に立ち向かっているのかを参考にすれば糸口が見えるはずです。

以下は Java のアプリケーションフレームワークである [Spring Test](https://spring.pleiades.io/spring-framework/docs/current/reference/html/testing.html#testing-tx) からの引用です。

> 実際のデータベースにアクセスするテストの一般的な課題の 1 つは、永続ストアの状態への影響です。開発データベースを使用する場合でも、状態の変更は将来のテストに影響を与える場合があります。
>
> (中略)
>
> TestContext フレームワークはこの課題に対処します。デフォルトでは、フレームワークは各テストのトランザクションを作成してロールバックします。トランザクションの存在を想定できるコードを作成できます。テストでトランザクション的にプロキシされたオブジェクトを呼び出す場合、設定されたトランザクションのセマンティクスに従って、それらは正しく動作します。さらに、テスト用に管理されているトランザクション内で実行中にテストメソッドが選択したテーブルの内容を削除すると、デフォルトでトランザクションがロールバックされ、データベースはテスト実行前の状態に戻ります。

Ruby における [RSpec Rails](https://relishapp.com/rspec/rspec-rails/docs/transactions) の `use_transactional_fixtures` もこれと同じアイディアですね。

### jest-prisma

前置きが長くなりましたが、Jest + Prisma でもこれらのフレームワークと同じ様に「テストケースごとにトランザクションを分離してテスト完了時にロールバック」を実現したく、作ったのが jest-prisma です。

https://github.com/Quramy/jest-prisma

jest-prisma は Jest の Test Environment です。

```js
/* jest.config.mjs */
export default {
  // ... Your jest configuration

  testEnvironment: "@quramy/jest-prisma/environment"
};
```

`jestPrisma.client` に jest-prisma に管理された Prisma Client のインスタンスが格納されます。簡単に利用例を示すと以下のようになります。

```ts
describe(UserService, () => {
  // jestPrisma.client works with transaction rolled-back automatically after each test case end.
  const prisma = jestPrisma.client;

  test("Add user", async () => {
    const createdUser = await prisma.user.create({
      data: {
        id: "001",
        name: "quramy"
      }
    });

    expect(
      await prisma.user.findFirst({
        where: {
          name: "quramy"
        }
      })
    ).toStrictEqual(createdUser);
  });

  test("Count user", async () => {
    expect(await prisma.user.count()).toBe(0);
  });
});
```

上記の例では、 `prisma.user` をそのまま Assertion していますが、実際のテストではテストケースの中でアプリケーションロジックの呼び出しが行われます。

### Interactive Transaction

jest-prisma は Prisma の [Interactive Transaction Feature](https://www.prisma.io/docs/concepts/components/prisma-client/transactions#interactive-transactions) を使っています。

```ts
prisma.$transaction(async clientInTransaction => {
  // この中の処理が一連のトランザクションとして扱われる

  // コールバックが reject されるとトランザクションがロールバックされる
  throw new Error();
});
```

この仕組を利用して `beforeEach` でトランザクションを発行して `afterEach` で例外投げてロールバックを引き起こすことで実現しています。

`beforeEach` や `afterEach` 的な仕組みがあれば Jest でなくても同じことができると思います。実際、@aiji42_dev さんが jest-prisma の仕組みを一部流用して vitest 用の Environment を作成してくれています。

https://github.com/aiji42/vitest-environment-vprisma

## テストデータのセットアップを書きやすくする

### 背景

jest-prisma で `use_transactional_fixtures` 相当を手に入れたはいいものの、実際にテストコードを量産しだしてみると、次の問題が浮上してきます。

それは「テストの事前状態を準備するのが面倒くさい」ということです。

例えば、テストしたい対象の Scehma が以下であったとします。GraphQL のサンプルでよくある「ブログエントリと著者」の構造です。

```graphql
model User {
  id    String @id
  email String @unique
  name  String
  posts Post[]
}

model Post {
  id       String @id
  title    String
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}
```

「Post model の id を指定して更新する」という更新系処理のテストを想定すると、当然「Post が存在している場合」というコンテキストが出てきます。これを素の Prisma Client で記述すると以下のようになるでしょう。

```ts
await prisma.post.create({
  data: {
    id: "post001",
    name: "sample blog post",
    author: {
      create: {
        data: {
          id: "user001",
          name: "Quramy",
          email: "Quramy@myservice.com"
        }
      }
    }
  }
});
```

「id が `post001` である Post を一件用意したい」というだけでこのザマです。もちろん、User Model の必須フィールドが増えれば増えるほど上記のコードの行数は膨れていきます。

このようなコードをテストケース毎に何度も何度も書くのは正直御免被りたいものです。

### FactoryBot 的に利用できるサムシング

@seya さんが同じ課題感に対して記事を書いています。

https://zenn.dev/seya/articles/a0d2d2da20ddad

上記の記事からの引用になりますが、実現したいことはまさに以下のイメージです。

```ts
// User の必須な値にはデフォルト値が生成されて入っている -> 全ての値を指定する必要がない
const user = await UserFactory.create({
  name: "John Doe"
});
```

同じく @seya さんが[別の記事](https://zenn.dev/seya/articles/5d384daafb1c24) で言及されていますが、 Rails でいうところの factory_bot に相当する何かが欲しいわけです。

そこで、Prisma で同じようなことが実現するために作成したライブラリが prisma-fabbrica です。ちなみに "fabbrica" はイタリア語で「工場(= factory)」の意です。"prisma" 自体がイタリア語というのもあるのですが、既に "prisma-factory" という npm パッケージが存在しており、他の名前を探した結果です。

https://github.com/Quramy/prisma-fabbrica

prisma-fabbrica は Prisma のジェネレータです。

```graphql
generator client {
  provider = "prisma-client-js"
}

generator fabbrica {
  provider = "prisma-fabbrica"
}
```

`schema.prisma` ファイルを上記のように設定しておき `npx prisma generate` のように実行するだけです。 生成されるものは「Factory を定義するための関数群」です。

生成された関数を利用して、以下のように Model 毎のファクトリを定義します。

```ts
/* src/factories.ts */

import { defineUserFactory, definePostFactory } from "./__generated__/fabbrica";

export const UserFactory = defineUserFactory();

export const PostFactory = definePostFactory({
  defaultData: {
    author: UserFactory
  }
});
```

テストコード側からは次のように利用します。 先程の「id が `post001` である Post を一件用意したい」であれば、次の一行に短縮されます。

```ts
await PostFactory.create({ id: "post001" });
```

`post.title` などの未指定な Scalar Field は自動で補完されますし、User -> Post の リレーションも `definePostFactory` した際に `UserFactory` をデフォルト挙動として指定しておくことで、テストコード側から意識しなくて済むようになっています。

上記の例では ID を明示的に指定していますが、省略した場合は Short UUID で補完されます。

fabbrica の提供するファクトリは `create` や `connectOrCreate` に渡せるオブジェクトを作っているだけなので、素の Prisma Client と同様にリレーションに `create` や `connectOrCreate` を指定することもできます。

```ts
// 指定した著者情報をもつ Post を作成
await PostFactory.create({
  author: { create: await UserFactory.build({ id: "user001" }) }
});

// Post を 3 件もつ User を作成
await UserFactory.create({
  posts: { create: await PostFactory.buildList(3) }
});
```

### Why generator

何故ジェネレータとして用意したのか、という点についてですが、これには２つの理由があります。

一点目が「`@prisma/client` は TypeScript から汎化して扱うことに不向き」です。

例えば `prisma.user.create` と `prisma.post.create` は同じ `create`というメソッドを持っていますが、中身の型定義は完全な別物です。仮に以下のような共用型合成をしたところで `CreateFn` は呼び出すことのできない関数型でしかありません。

```ts
// どうやっても呼び出すことのできない関数型
type CreateFn = (Prisma.UserDelegate<any> | Prisma.PostDelegate<any>)["create"];
```

これは prisma-client-js ジェネレータによって、それぞれの Model 毎に `create` が別個のメソッド定義として生成されているためです。
要するに コード自動生成の上にエコシステムを構築していくのが Prisma way ということなので、prisma-fabbrica も「 Model 毎のヘルパ」を生成するようにしています。

二点目は「Schema 情報にアクセスするため」です。 Factory ヘルパの構築にあたり、例えば「どの Field に一意制約が課せられているのか」といった情報が必要になります。
Prisma ジェネレータは Prisma CLI から起動される際に DMMF(Data Model Meta Format) という構造体が渡されます。DMMF はその名のとおり、`schema.prisma` ファイルが保持しているメタ情報を格納しており、以下のような情報にアクセスすることができます。

- 各 Model 間のリレーション情報
- `@default` や `@@id` など、 Model に付与されたメタ情報

prisma-fabbrica でも DMMF から抽出した情報を元に、Model 毎の Factory 用ヘルパ関数のコードを生成するようにしています。

（補記） このエントリの下書きを書いている最中に Prisma 4.7 がリリースされ、[Client extensions](https://www.prisma.io/docs/concepts/components/prisma-client/client-extensions) という Preview feature が登場しました。「コード自動生成の上にエコシステムを構築していくのが Prisma way」と書きましたが、Client extensions を利用すると動的な Prisma Client の拡張が行えるため、もしかすると今後は Code Generation だけではなく、Client extensions による機能拡張が主流になっていくかもしれません。

### その他の機能

このエントリでは詳細は触れませんが、prisma-fabbrica には以下のような機能も備えています。

- Scalar 値の自動補完ルールを個別に設定する
- jest-prisma との連携
- etc,,,

また、まだ検討中ですが、Trait や Transient Attribute などの factory_bot に備わっている機能も必要に応じて取り込んでいけたらと思っています。

## おわりに

このエントリでは Prisma の Integration Test を補助するツールとして jest-prisma / prisma-fabbrica を紹介しました。まだ公開して日が浅いですが、自分でも使い込みつつブラッシュアップできたらと思います。興味があればぜひ利用してもらえると嬉しいです。

また、以下のレポジトリに 紹介したライブラリ双方を使ったテストのサンプルを置いています。こちらも参考にしていただければと。

https://github.com/Quramy/prisma-yoga-example/tree/main/tests
