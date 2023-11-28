# SuperTest と msw 併用時に warning log を抑止する

BFF などの「Upstream Service に HTTP を叩いてその response を加工しつつ、自分自身の response を返却する」類の Node.js API サーバーを考える。

Service としてのスタックは以下のようなイメージ:

- API Aggregation Endpoint: Express
- Upstream HTTP Call: Fetch API (cross-fetch, undici, etc...)

この Service に対して、API Integration Test を記述する目的で以下を導入したとする。

- jest
- SuperTest
- `msw/node`

すなわち、API Aggregation Endpoint の実装が下記のようになっていて、

```ts
import express from "express";

export const app = express();

app.get("/api/foo", (req, res) => {
  fetch("http://upstream.service.local/api/hoge")
    .then(result => {
      if (!result.ok) return res.status(400).end();
      return result.json() as { value: number };
    })
    .then(data => {
      return res
        .json({ result: data.value * 2 })
        .status(200)
        .end();
    });
});
```

これに対する Integration Test が下記のようになっているイメージ。

```ts
import request from "supertest";
import { rest } from "msw";
import { setupServer } from "msw/node";

import { app } from "./app";

describe("GET /api/foo", () => {
  const upstreamServices = setupServer(
    ...[
      rest.get("http://upstream.service.local/api/hoge", (req, res, ctx) =>
        res(ctx.json({ value: 100 }), ctx(200))
      )
    ]
  );

  beforeAll(() => upstreamServices.listen());
  afterEach(() => upstreamServices.resetHandlers());
  afterAll(() => upstreamServices.close());

  it("should respond 200", async () => {
    const response = await new Promise<request.Response>((resolve, reject) =>
      request
        .get("/api/foo")
        .expect(200)
        .end((err, response) => (err ? reject(err) : resolve(response)))
    );
    expect(response.body.result).toBe(200);
  });
});
```

上記のパターンの integration test 記述を初めてやったのだけど、msw と SuperTest を併用していると、msw 側が SuperTest の対象の request について「それ、mock handler すり抜けるけど大丈夫？」的な Warning を jest に履いてきて地味に鬱陶しい。

```text
  console.warn
    [MSW] Warning: captured a request without a matching request handler:

      • POST http://127.0.0.1:36489/api/foo/

    If you still wish to intercept this unhandled request, please create a request handler for it.
    Read more: https://mswjs.io/docs/getting-started/mocks
```

`listen` するときに `onUnhandledRequest` というオプションが用意されていて、ここを引っ掛けると柔軟にカバーできる。

大概の場合、SuperTest に対する Subject Endpoint だけを Warning から除外すれば十分なはずなので、以下のようにすると良さそう。

```ts
upstreamServices.listen({
  onUnhandledRequest: (mockedReq, print) => {
    if (mockedReq.url.pathname === "/api/foo") return null;
    print.warning();
  }
});
```
