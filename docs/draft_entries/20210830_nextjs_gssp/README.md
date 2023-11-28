## Next.js + Express での getServerSideProps

最近、おしごとで Next.js on Express な構成に触れることが多い。

Express、すなわち自前サーバーを持っていると「サーバーサイドで色々ゴニョゴニョしてから画面を描画したい」という要求は割と色々出てくる。
こういうときに使うのは皆大好き `getServerSideProps` で、今日はその部分のちょっとした tips 的な話。

レンダリング前に何かしらのサーバーサイド処理をはさみたい場合、Next.js では以下のように書く。

```ts
/* src/pages/Hoge.tsx */

import { GetServerSideProps } from "next/core";

export function Page() {
  return; // ページ部分
}

export const getServerSideProps: GetServerSideProps = async ({
  req,
  query
}) => {
  // do something
  return {
    props: {
      // Properties for App or Page
    }
  };
};
```

これですごい困る、という程ではないのだけど、いくつか鬱陶しいところがある。

- Express と併用している場合、 `req` や `res` の型変換が面倒。express middleware で追加したプロパティにアクセスするために、都度型変換が必要。
- 「ページを描画する前にサーバーサイドで実行したい処理」を分離して管理するのが若干面倒。

なので、以下のようなユーティリティ関数 `combineGssp` をプロジェクトに突っ込んでおくことが多い。

```ts
/* src/functions/combineGssp.ts */

import { Request, Response } from "express";

import type { ParsedUrlQuery } from "querystring";
import {
  GetServerSideProps,
  GetServerSidePropsContext,
  GetServerSidePropsResult
} from "next";

export type ExtendedGetServerSidePropsContext<
  Q extends ParsedUrlQuery = ParsedUrlQuery
> = GetServerSidePropsContext<Q> & {
  req: Request;
  res: Response;
};

export type ExtendedGetServerSideProps<
  P extends { [key: string]: any } = any,
  Q extends ParsedUrlQuery = ParsedUrlQuery
> = (
  ctx: ExtendedGetServerSidePropsContext<Q>
) =>
  | Promise<GetServerSidePropsResult<P> | undefined>
  | GetServerSidePropsResult<P>
  | undefined;

function isPropsTypeResult(
  x: GetServerSidePropsResult<any>
): x is { props: any } {
  return (
    typeof x === "object" &&
    (x as any)["props"] &&
    typeof (x as any).props === "object"
  );
}

export function combineGssp<
  P extends Record<string, any> = Record<string, any>
>(...chain: ExtendedGetServerSideProps<any, any>[]) {
  const fn: GetServerSideProps<P> = async ctx => {
    let mergedProps: any = {};
    for (const gsspFn of chain) {
      const result = await gsspFn(ctx as ExtendedGetServerSidePropsContext);
      if (!result) continue;
      if (!isPropsTypeResult(result)) {
        return result;
      }
      mergedProps = {
        ...mergedProps,
        ...result.props
      };
    }
    return {
      props: mergedProps
    };
  };
  return fn;
}
```

いきなり `combineGssp` の型定義を見ると複雑そうに見えるが基本的には Next.js そのものの `GetServerSideProps` を少し拡張した関数をいくつか用意し、その組み合わせで `getServerSideProps` を構成しよう、という考え方。

例を見せた方がわかりやすいと思う。

次のコードは下記の処理を個別に用意しておき、それらを合成して `getServerSideProps` 関数にする例。

- セッションにデータが無いときは ログイン画面にリダイレクト
- 画面用のデータを事前取得する (ホントは swr や react-query の cache dehydration 系の処理になったりするけど、ここでは雑に書いてる)

```ts
/* src/pages/Hoge.tsx */

import {
  combineGssp,
  ExtendedGetServerSideProps
} from "../functions/combineGssp";

export function Page() {
  return; // ページ部分
}

export const loginProtection: ExtendedGetServerSideProps = ({ req }) => {
  if (!req.session?.user) {
    return {
      redirect: {
        destination: "/login"
      }
    };
  }
};

export const prefetchHoge: ExtendedGetServerSideProps = async ({ req }) => {
  const { data } = await req.someMiddlewareFunction();
  return {
    props: {
      hogeData: data
    }
  };
};

export const getServerSideProps = combineGssp(loginProtection, prefetchHoge);
```
