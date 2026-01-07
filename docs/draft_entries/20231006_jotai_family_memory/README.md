# Jotai の Atom Family と メモリリーク対策

最近、Next.js (App Router) と [Jotai](https://jotai.org) を組み合わせて使っている。

Jotai は Atomic な State Management ができる alt Recoil なライブラリが欲しかったたために導入した。

基本的な使い心地は満足しているが、Atom Family を使うときだけは注意が必要なので、その備忘。

https://jotai.org/docs/utilities/family#caveat-memory-leaks に記載がある通り、Jotai の Atom Family は `Map` で実装されているため、サーバーサイドで無邪気に利用するとメモリリークしていく。

> Internally, atomFamily is just a Map whose key is a param and whose value is an atom config. Unless you explicitly remove unused params, this leads to memory leaks

以下のような Client Component をそのまま Node.js で動かしてはいけない、という意味。

```jsx
import { atom, useAtom } from "jotai";
import { atomFamily } from "jotai/utils";

const myAtomById = family(() => atom(false));

export default function MyComponent({ id }) {
  const [value, setValue] = myAtomById(id);

  // 以下略
}
```

Jotai のドキュメントにもある通り、 明示的に `remove` または `setShouldRemove` を呼び出す必要がある。例えば以下のようにすれば、作成から 10 秒経った Family に属する Atom を全削除できる。

```js
const now = Date.now();
myAtomById.setShouldRemove((createdAt) => now - createdAt > 10_000);
```

App Router と併用する場合、上記のコードをサーバーサイドのどこかに仕込む必要がある。最初は kubernetes の Liveness Prove 辺りに仕込んでしまえばいいか？と思ったが、 Route Handler に上記のコード書いたら、 Next.js Compiler に怒られたのでボツ案。

```jsx
import { atom, useAtom } from "jotai";
import { atomFamily } from "jotai/utils";

const myAtomById = family(() => atom(false));

export default function MyComponent({ id }) {
  const [value, setValue] = myAtomById(id);

  if (typeof window === "undefined")
// Client Component が
    const now = Date.now()
    myAtomById.setShouldRemove((createdAt) => now - createdAt > 10_000);
  }

  // 以下略
}
```
