# Remix with Express と Dev Server

Remix において標準の `remix-serve` コマンドをイジェクトして自前の Server を利用したくなることがある[^1]。

[Remix のドキュメント](https://remix.run/docs/en/main/other-api/adapter) にある通り、`@remix-run/express` から `createRequestHandler` を 持ってきて Express のミドルウェアとして噛ませればカスタムサーバを作ることはできる。

```js
const { createRequestHandler } = require("@remix-run/express");
const express = require("express");

const app = express();

// needs to handle all verbs (GET, POST, etc.)
app.all(
  "*",
  createRequestHandler({
    // `remix build` and `remix dev` output files to a build directory, you need
    // to pass that build to the request handler
    build: require("./build"),

    // return anything you want here to be available as `context` in your
    // loaders and actions. This is where you can bridge the gap between Remix
    // and your server
    getLoadContext(req, res) {
      return {};
    },
  })
);
```

上記のコードサンプルを見ても明らかな通り、これは Production Build した成果物がある前提でのコードであって、Dev Server (`remix dev`) のことを考えると不十分。たとえば `getLoadContext` に値を詰めたとしても、それが `npm run dev` の場合に参照できないと不便だ（もちろん、特に Remix の req/res を拡張せずにアプリケーションが組めるのであればそれが理想なのだけど）。

要するに、`remix-serve` だけじゃなくて `remix dev` もイジェクトしたくなるわけだけど、軽くドキュメントを探してもあまり有用な情報が見つからずに少し苦労したため、現状で自分が利用している Remix Express 用の Dev Server を備忘代わりに置いておく。

```js
/* server.mjs */

import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import express from "express";
import remix from "@remix-run/express";

const __dirname = dirname(fileURLToPath(import.meta.url));

async function createDevServer() {
  const vite = await import("vite");
  return vite.createServer({
    root: __dirname,
    configFile: resolve(__dirname, "./vite.config.ts"),
    server: {
      middlewareMode: true,
      hmr: { host: "localhost", port: 10001 },
    },
  });
}

const mode = process.env.NODE_ENV;
const port = process.env.PORT ?? (mode === "production" ? 3000 : 5173);

const app = express();

const devServer = mode !== "production" ? await createDevServer() : undefined;

const handleClientAssets = devServer
  ? devServer.middlewares
  : express.static(resolve(__dirname, "./build/client"));

const handleSSR = remix.createRequestHandler({
  build: devServer
    ? () => devServer.ssrLoadModule("virtual:remix/server-build")
    : await import("./build/server/index.js"),
});

app.all("*", handleClientAssets, handleSSR);

app.listen(port, () => {
  console.log(`Listening on http://localhost:${port}`);
});
```

`remix dev` や `remix-serve` の代わりに `node server.mjs` として動く。

[^1]: 自分の場合、 SSR 時に Loader と Render で共通で参照できるリクエストローカルなスコープとして、Async Local Storage が欲しくなった、という切欠。素の `remix dev` のままで実現できる方法があればむしろそれを知りたい。
