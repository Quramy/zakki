# Storybook で Apollo Client の `useFragment` を扱う

Apollo Client 3.8 から `useFragment` という React hook が導入されました。

https://speakerdeck.com/quramy/apollo-client-usefragment?slide=8

今回はこの `useFragment` を利用している Component を対象としたテストや Storybook の書き方についての覚書です。

先日に別のエンジニアから、Storybook の記述に際して相談を受けた際のやり取りを元ネタとしています。

なお、途中のコードが長くなりそうなため、先に記事で取り扱いたいトピックを挙げておきます。

- `useFragment` を利用している Component の Storybook が記述できるようになること
- Storybook で扱う Fragment のスタブデータの構築に一定の柔軟性を持たせること

各トピックに対応する解として、この記事では以下の方式について解説します。

- Apollo Cache を用意し、Story 描画前に所望の状態となるようにする
- graphql-codegen-typescript-fabbrica でスタブデータのファクトリを用意する

また、先に前提を書いておきますが、GraphQL Fragment と React Component は Colocate で管理されている想定です。Colocate ナニソレという人は以下を読んでください。

https://speakerdeck.com/quramy/fragment-composition-of-graphql

## Apollo Cache を用意し、Story 描画前に所望の状態となるようにする

さて、 Apollo Client の `useFragment` を利用した Component は以下のようになっているとします。

```tsx
/* src/components/Avatar/index.tsx */

import { useFragment } from "@apollo/client";
import { graphql } from "@/gql";

export const fragment = graphql(`
  fragment Avatar_User on User {
    name
    avatarURL
  }
`);

export type Props = {
  readonly id: string;
};

export function Avatar({ id }: Props) {
  const { complete, data: user } = useFragment({
    fragment,
    fragmentName: "Avatar_User",
    from: {
      __typename: "User",
      id,
    },
  });

  if (!complete) return null;

  return (
    <>
      <img width={48} height={48} src={user.avatarURL} alt={user.name} />
    </>
  );
}
```

上記の `Avatar` Component について、いつものように CSF で Story を書くと以下のようになります。

```tsx
/* src/components/Avatar/index.stories.tsx */

import type { Meta, StoryObj } from "@storybook/react";

import { Avatar } from ".";

const meta = {
  title: "components/Avatar",
  component: Avatar,
  args: {
    id: "user001",
  },
} satisfies Meta;

export default meta;

type Story = StoryObj<typeof meta>;

export const Default = {} satisfies Story;
```

当然ですが、このままでは正しく動作しません。

`user001` の Fragment データを注入していないというのもありますが、それ以前に Story に `ApolloClient` のインスタンスが提供されていません。

Apollo Client に限らず、 React Context Provider と連動する Story を記述する際の定番は Decorator です。

```tsx
/* src/support/storybook/apollo.tsx */

import { Decorator } from "@storybook/react";
import { ApolloProvider, ApolloClient } from "@apollo/client";

const apolloDecorator: Decorator = (Story, ctx) => {
  const client = new ApolloClient({
    /* Storybook 用の Apollo Client 設定 */
  });

  return (
    <ApolloProvider client={client}>
      <Story />
    </ApolloProvider>
  );
};
```

ところで、Apollo Client には [MockedProvider](https://www.apollographql.com/docs/react/development-testing/testing/#the-mockedprovider-component) というテスト用の Provider も用意されていますが、結局のところ柔軟性に欠ける代物なので、今回の記事では取り扱いません。

作成した Decorator は preview.tsx に設定しておきます（個別の CSF に都度設定してもよいのですが、面倒なので一括で設定しています）。

```tsx
/* .storybook/preview.tsx */

import type { Preview } from "@storybook/react";
import { apolloDecorator } from "@/support/storybook/apollo";

const preview: Preview = {
  /* 略 */
  decorators: [apolloDecorator],
};

export default preview;
```

今回は .stories ファイルの側から 任意の Fragment の状態を用意してあげたいので、Apollo Client 構築時にわたす `cache` オプションを渡せるように考えてみます。

```ts
const apolloDecorator: Decorator = (Story, context) => {
  // context には Story の定義情報 (parameterなど) が詰まっている
  const cache = getCacheFrom(context);

  const client = new ApolloClient({
    cache,
  });
  // 略
};
```

CSF から上記の Decorator に何かしらのデータを受け渡すために利用できる口としては `parameters` または `loaders` がありますが、今回は `loaders` を選択します。
`loaders` は、Story を描画するよりも前に実行する非同期関数を設定するために用います。今回は Fragment スタブデータの構築に用いますが、Next.js における `getServerSideProps` や Remix における `loader` 関数などを代替させたいようなケースでは便利と思います。

```tsx
/*  myStory.stories.tsx */
const meta = {
  loaders: async () => {
    const stubData = await buildStubData();
    return {
      stubData,
    };
  },
} satisfies Meta;
```

ここまでの設計は次のようになります:

- `ApolloClient` を供給する Storybook Decorator を用意する
- Decorator 用の Apollo Cache を構築する Loader を用意する

Loader のコードはここに書くと流石に煩雑なので、興味がある人は以下のリンク先を見てください。
基本的には、 Apollo Cache の `writeFragment` 関数を使っているだけです。

https://github.com/Quramy/apollo-client-storybook-example/blob/487bf74b82050617e813631f5044e5d4d1b3d24e/src/support/storybook/apollo.tsx#L85

これによって、以下のように `loaders` で 対象の Component が`useFragment` する際のデータを注入できるようになりました。

```tsx
/* src/components/Avatar/index.stories.tsx */

import type { Meta, StoryObj } from "@storybook/react";

import { createCachePreloader } from "@/support/storybook/apollo";

import { Avatar, fragment } from ".";

const meta = {
  title: "components/Avatar",
  component: Avatar,
  loaders: createCachePreloader()
    .preloadFragment({
      fragment,
      fragmentName: "Avatar_User",
      data: {
        __typename: "User"
        id: "user001",
        name: "Quramy",
        avatarURL: "https://fakeimg.pl/48x48/23cd6b/fff",
      },
    })
    .toLoader(),
  args: {
    id: "user001",
  },
} satisfies Meta;

export default meta;

type Story = StoryObj<typeof meta>;

export const Default = {} satisfies Story;
```

## graphql-codegen-typescript-fabbrica でスタブデータのファクトリを用意する

ここまでで Loader と Decorator で 任意の Fragment Data を Story に差し込めるようになりました。

後はこの仕組みを使って Story をガリガリと書いていけばいいのですが、まだ課題があったりします。

端的に言うと、スタブデータ用意するのが面倒問題というやつです。

GraphQL Client で Fragment と Component を Colocate している場合、Root(Query) に近い Component は、自身から見て子・孫となる Component が必要とする Fragment のデータに対しても（推移的に）依存していることになります。

例として、上述で例としていた `Avatar` Component に依存している `PostSummary` Component を考えます。

https://github.com/Quramy/apollo-client-storybook-example/blob/9fdb06ccf25db6b9684c8e5614f1a96d4b4b8079/src/components/PostSummary/index.tsx

この `PostSummary` Component の fragment 定義は以下のようになっており、この Fragment に相当するデータを用意するということはすなわち、子階層の `Avatar_User` の Fragment 相当のデータまで用意することを意味します。

```gql
fragment PostSummary_Post on Post {
  id
  title
  description
  author {
    id
    name
    ...Avatar_User
  }
}
```

言ってしまえば「推移的に関連している先の構造のデータを一々用意したくない」「なるべく楽にスタブデータを構築したい」という話です。

この件については @mizdra さんが似たようなことを以下のブログで書いてくれています。

https://www.mizdra.net/entry/2023/09/28/174037

> ただ、このようにモックデータをベタ書きしていくと、コードの重複が増えていきます。
>
> (中略)
>
> 一般にはこうした問題を回避するために、type 単位のダミーレスポンスを作成する補助関数 (factory みたいなやつ) を自作している人が多いんじゃないかと思います。
>
> (中略)
>
> これで幾分か楽になりますが、factory 関数を時前で実装したりメンテナンスしていくのは手間です。また、「id をオートインクリメントしつつモックデータを作成したい」「Book を N 個詰め込んだ配列を作って欲しい」など等色々な要件が出てくると、factory 関数の実装が複雑になってきます。

このブログとともに紹介されているのが、 https://github.com/mizdra/graphql-codegen-typescript-fabbrica というユーティリティライブラリです。

光栄なことに自分の名前もクレジットしてもらっていたこともあり、以前から「その内機会があったら使ってみよう」とは思っていたため、今回の相談のタイミングで試してみました。

graphql-codegen-typescript-fabbrica の導入自体は README 通りに行なうだけなので、特に困ることは無いため記載は省略します。

まずは Leaf(末端) 側である `Avatar` Component の Story で graphql-codegen-typescript-fabbrica を利用するように改修してみました。

```tsx
/* src/components/Avatar/index.stories.tsx */

import type { Meta, StoryObj } from "@storybook/react";

import { createCachePreloader } from "@/support/storybook/apollo";
import { defineUserFactory, dynamic } from "@/__generated__/fabbrica";

import { Avatar, fragment } from ".";

export const UserFragmentFactory = defineUserFactory({
  defaultFields: {
    __typename: "User",
    id: dynamic(({ seq }) => `user${seq}`),
    name: "Quramy",
    avatarURL: "https://fakeimg.pl/48x48/23cd6b/fff",
  },
});

const meta = {
  title: "components/Avatar",
  component: Avatar,
  excludeStories: /Factory$/, // CSF で直接 Factory を export しているため
  loaders: createCachePreloader()
    .preloadFragment({
      fragment,
      fragmentName: "Avatar_User",
      data: UserFragmentFactory.build({ id: "user001" }),
    })
    .toLoader(),
  args: {
    id: "user001",
  },
} satisfies Meta;

export default meta;

type Story = StoryObj<typeof meta>;

export const Default = {} satisfies Story;
```

`UserFragmentFactory` を export しているのは、この `Avatar` Component(すなわち `Avatar_User` Fragment) に依存している Component の Story を書くときに再利用したいためです。

たとえば、先ほどの例と同じく `PostSummary` Component が `Avatar` に依存しており、以下の Fragment を使っていたとします。

```gql
fragment PostSummary_Post on Post {
  id
  title
  description
  author {
    id
    name
    ...Avatar_User
  }
}
```

この `PostSummary` Component に相当する Story と、ここで必要となる Fragment のファクトリは以下のように記述できます。

```tsx
/* src/components/PostSummary/index.tsx */

import type { Meta, StoryObj } from "@storybook/react";

import { createCachePreloader } from "@/support/storybook/apollo";
import { definePostFactory, dynamic } from "@/__generated__/fabbrica";

import { UserFragmentFactory } from "../Avatar/index.stories";

import { PostSummary, fragment } from ".";

export const PostFragmentFactory = definePostFactory.withTransientFields({
  authorName: "",
})({
  defaultFields: {
    __typename: "Post",
    title: dynamic(({ seq }) => `Awesome blog post ${seq}`),
    description: "description description",
    id: dynamic(({ seq }) => `post${seq}`),
    author: dynamic(
      async ({ get }) =>
        await UserFragmentFactory.build({
          name: (await get("authorName")) || "hogefuga",
        })
    ),
  },
});

const meta = {
  title: "components/PostSummary",
  component: PostSummary,
  excludeStories: /Factory$/,
  loaders: createCachePreloader()
    .preloadFragment({
      fragment,
      fragmentName: "PostSummary_Post",
      data: PostFragmentFactory.build({
        id: "post001",
        title: "Apollo Client with Storybook",
        authorName: "Quramy",
      }),
    })
    .toLoader(),
  args: {
    id: "post001",
  },
} satisfies Meta;

export default meta;

type Story = StoryObj<typeof meta>;

export const Default = {} satisfies Story;
```

Factory を定義する際に TypeScript の恩恵が受けられる、という部分もさることながら、以下が実現されている点が個人的には気に入っています。

- スタブデータのファクトリ関数も Fragment Tree (Component 階層) と同じ依存関係を保っている。 `Avatar` でしか使っていない `avatarUrl` フィールドについて、親 Component からの不可知なまま
- 親と子で共通で利用することになる `User.name` フィールドは、親側でオーバーライドすることでコントールできている
- Transient Field で `User.name` をさらに外から制御する口(`authorName`) を設けることもできる

こんな感じで Component - Fragment - Story - Factory のセットをまとめていくと、最終的には `useQuery` (または `useSuspenseQuery` ) を実行する Component (一般的には Page Component) に辿り着きます。

Apollo Cache におけるメソッドこそ、`writeFragment` ではなく `writeQuery` に変更する必要はありますが、ここまでと同じ考え方で `Query` type に対するファクトリと `createCachePreloader` で 最上位 Component についても同様に Story を記述できます。

https://github.com/Quramy/apollo-client-storybook-example/blob/9fdb06ccf25db6b9684c8e5614f1a96d4b4b8079/src/components/PopularPosts/index.stories.tsx

## おわりに

今回の記事では、以下のトピックを中心として、 Storybook の Decorator / Loaders の扱い方や graphql-codegen-typescript-fabbrica を用いた効率的なスタブデータの構築方法について記載しました。

- `useFragment` を利用している Component の Storybook が記述できるようになること
- Storybook で扱う Fragment のスタブデータの構築に一定の柔軟性を持たせること

特に Storybook Decorator を経由させて React Component に Context を与えるパターンは、Apollo Client に限らず有用な手段であるため、必要に応じてプロジェクトに即した Decorator を用意できるようにしておくと、テストでできることの幅が広がっていくと思います。

なお、文中でも何度か参照していますが、今回の記事で用いているソースコードは以下のレポジトリに格納しています。

https://github.com/Quramy/apollo-client-storybook-example

## おまけ 1: Unit Test でも Story を再利用する

Storybook を活用しているプロダクトであれば、Jest や Vitest で `composeStory` 関数と testing-library を組み合わせ、 CSF を再利用しているケースも多いと思います。
通常だと、以下のように利用しているはず。

```ts
/* src/components/Avatar/index.test.tsx */

import { render, screen } from "@testing-library/react";
import { composeStory } from "@storybook/react";
import Meta, { Default } from "./index.stories";

describe(Meta.title, () => {
  test("render fragment", () => {
    // Arrange
    const Component = composeStory(Default, Meta);

    // Act
    render(<Component />);

    // Assert
    expect(screen.getByRole("img")).toBeInTheDocument();
  });
});
```

残念なことに、このテストは期待通りに動作しません。
`composeStory` は Story に記載されている `loaders` の面倒までは見てくれないためです。
仕方ないので、 Story の `loaders` Annotation に記載されている非同期関数を `composeStory` より事前に実行しておき、取得できた結果を `parameters` を媒介して受け渡すようにしています。

```ts
/* src/components/Avatar/index.test.tsx */

import { render, screen } from "@testing-library/react";
import { composeStory } from "@storybook/react";
import { preloadStory } from "@/support/storybook/testing";
import Meta, { Default } from "./index.stories";

describe(Meta.title, () => {
  test("render fragment", async () => {
    // Arrange
    const loaded = await preloadStory(Default, Meta); // `loaders` を実行し、結果を parameters に受け渡しておく
    const Component = composeStory(Default, Meta, {
      parameters: {
        ...loaded,
      },
    });

    // Act
    render(<Component />);

    // Assert
    expect(screen.getByRole("img")).toBeInTheDocument();
  });
});
```

上記における `preloadStory` が `loaders` Annotation に記載された非同期関数を実行する関数です。

https://github.com/Quramy/apollo-client-storybook-example/blob/94a2c2e7d412e45388ddef4640f6066d0b9effea/src/support/storybook/testing.ts

これに付随して、Decorator 側でも `context.loaders` と `context.parameters` の両方から Apollo Cache を取り出すようにしています。

https://github.com/Quramy/apollo-client-storybook-example/blob/94a2c2e7d412e45388ddef4640f6066d0b9effea/src/support/storybook/apollo.tsx#L66-L76

Story 側に記載された Fragment Stub をそのまま使うのではなく、明示的にテストケースで事前条件とする Fragment 情報を明記した方が都合のよいケースであれば、以下のように `createCachePreloader` を使うことで実現できます。

```ts
/* src/components/Avatar/index.test.tsx */

import { render, screen } from "@testing-library/react";
import { composeStory } from "@storybook/react";

import { createCachePreloader } from "@/support/storybook/apollo";
import { fragment } from ".";
import Meta, { Default, UserFragmentFactory } from "./index.stories";

describe(Meta.title, () => {
  test("alt attribute", async () => {
    // Arrange
    const loaded = await createCachePreloader()
      .preloadFragment({
        fragment,
        fragmentName: "Avatar_User",
        data: UserFragmentFactory.build({
          id: "test_user",
          name: "quramy",
        }),
      })
      .load();
    const Component = composeStory(
      {
        ...Default,
        args: {
          id: "test_user",
        },
      },
      Meta,
      {
        parameters: {
          ...loaded,
        },
      }
    );

    // Act
    render(<Component />);

    // Assert
    expect(screen.getByAltText("quramy")).toBeInTheDocument();
  });
});
```

## おまけ 2: Mutation のスタブ

`useQuery` や `useFragment` については「事前に Apollo Cache を暖機運転させておく」という考え方で割とどうにでもなるのですが、 `useMutation` に代表されるようなユーザーインタラクションが発生して初めて Cache の状態が変化する類の機能について、Story を書こうとすると、もう一手間書ける必要があります。

これについても、 Apollo Client を提供する Decorator にて Apollo Link を `SchemaLink` に差し替えるようにすれば、任意の Resolvers を Story に差し込めます。

```tsx
/* src/support/storybook/apollo.tsx */

import { Decorator } from "@storybook/react";
import { ApolloProvider, ApolloClient } from "@apollo/client";
import { SchemaLink } from "@apollo/client/link/schema";

const apolloDecorator: Decorator = (Story, ctx) => {
  const mockSchema = context.parameters.mockSchema;
  const link = mockSchema ? new SchemaLink({ schema: mockSchema }) : undefined;

  const client = new ApolloClient({
    /* Storybook 用の Apollo Client 設定 */
    link,
  });

  return (
    <ApolloProvider client={client}>
      <Story />
    </ApolloProvider>
  );
};
```

Apollo Client が用意している `MockedProvider` と似てはいますが、 `MockedProvider` の場合、Resolver 相当として渡すオブジェクトに若干の癖があるため、個人的には生の `ApolloProvider` を利用しつつ、スタブとして利用する Resolvers は graphql-tools の `makeExecutableSchema` などを利用する方が好みです。

https://the-guild.dev/graphql/tools/docs/api/modules/schema_src#makeexecutableschema

これも若干長めとなるので、リンクだけ貼っておきます。

https://github.com/Quramy/apollo-client-storybook-example/blob/94a2c2e7d412e45388ddef4640f6066d0b9effea/src/components/PostDetail/index.stories.tsx
