# Accessible なセレクタ

久しぶりに Puppeteer の話。

社内の技術共有会で https://github.com/puppeteer/recorder という npm パッケージの話が挙がったのだけど、ここで登場する ARIA Selector という機能が面白い内容だったため、それを書いていこうと思う。

https://developers.google.com/web/updates/2020/11/puppetaria と重複しそう。

## Puppeteer v5.4 で導入された ARIA handler について

puppeteer/recorder 自体は名前の通り、Headless Chrome ラッパである Puppeteer を使って Chromium での操作を記録し、記録内容を再実行可能な JavaScript コードとして吐き出だすためのツールだ。

下記のように CLI として起動すると、Puppeteer が起動するので、ポチポチ操作を行うだけでよい。

```sh
$ npx @puppeteer/recorder https://github.com/Quramy --output recorded.js
```

例として、次の GIF のような操作をしたとしよう。

![Recording GitHub navigation](recording_github.gif')

1. https://github.com/Quramy を開く
1. ts-graphql-plugin のレポジトリを選択する
1. 検索ボックスに "addon" と入力し、レポジトリ内検索を行う

この操作を行うと、puppeteer/recorder は以下のスクリプトを出力する。

```js
const {
  open,
  click,
  type,
  submit,
  expect,
  scrollToBottom
} = require("@puppeteer/recorder");
open("https://github.com/Quramy", {}, async page => {
  await click("aria/ts-graphql-plugin");
  expect(page.url()).resolves.toBe(
    "https://github.com/Quramy/ts-graphql-plugin"
  );
  await click("aria/Search");
  await type('aria/Search[role="textbox"]', "addon");
  await click("aria/Search addon in this repository");
  expect(page.url()).resolves.toBe(
    "https://github.com/Quramy/ts-graphql-plugin/search?q=addon"
  );
});
```

実のところ、このスクリプトを実行させても、（僕が試した今日現在では）全然まっとうに動かないし、expect の中身間違ってるわ、ちゃんと navigation を wait できてねーわではっきり言って使い物にならない。
現状でこの CLI の利用はこれっぽちもお勧めできないし、ただオートメーションを記録したいだけであれば、Katalon のようなツール使ったほうがいいと思う。

わざわざ今回取り上げたは、要素のセレクタの話をしたかったからだ。例えば下記の行。

```js
await click("aria/ts-graphql-plugin");
```

この行における `click` 関数は puppeteer/recorder が export しているものだけど、やっていることは Puppeteer を生で使った場合の下記と一緒だ。

```js
const handler = await page.waitForSelector("aria/ts-graphql-plugin");
await handler.click();
```

注目して欲しいのは、クリック対象の要素を選択するセレクタの文字列 `'aria/ts-graphql-plugin'` という部分。
セレクタ文字列 `aria/` prefix から始まると、デフォルトの CSS セレクタとしてではなく、ARIA Handler というクエリハンドラで検索されるように、カスタムハンドラが登録されている。これは Puppeteer v5.4 から。

このハンドラは、名前の通り、ARIA 情報に基づいて要素を特定する手段を提供してくれる。

先程の `page.waitForSelector("aria/ts-graphql-plugin")` であれば、「名前が"ts-graphql-plugin"である要素」という意味のセレクタになる。

`page.waitForSelector("[name='ts-graphql-plugin']")` とは意味が違う。ここでいう「名前が」とは、Accessible Name として、ということだ。

`page.waitForSelector("aria/ts-graphql-plugin")` がマッチする要素の例は、例えば下記だ。

```html
<img src="repo.png" alt="ts-graphql-plugin" />

<a href="./ts-graphql-plugin">ts-graphql-plugin</a>
```

もっと複雑な例はいくらでも考えられるし、 [Accessible Name の仕様](https://www.w3.org/TR/accname-1.1/#mapping_additional_nd_te)を見れば、同じことを DOM の API だけで実現するのがどれだけ面倒かが想像つくと思う。

ここで重要なのは、CSS セレクタに比べて、Accessible Name で要素を特定する方がよりセマンティックである、という点。

E2E や RPA の文脈で課題として挙がるのが、セレクタの保守性だ。

昨今は CSS クラス名は自動生成されていたりすることも多くビルドをし直したら CSS クラス名が変わってしまって以前マッチしたセレクタが動かなくなることはザラだろう。
また、BEM のような予測可能なクラス名のルールを使っていたとしても、E2E スクリプトが内部の実装に強く依存してしまう点は一緒だ。

[Autify blog なぜ E2E テストで id を使うべきではないのか](https://blog.autify.com/ja/why-id-should-not-be-used) に種々のセレクタの良し悪しについて詳細が紹介されていて、その中で下記のように書いてある。

> 文言をロケータに使うメリットは、先に述べたように 要素内のテキストが変更されたときにテストが失敗する点です。要素の外部的な振る舞いを保護できるとも言い換えられます。

要素中のテキストを検索する方法は幾つかあるけれど、CSS セレクタでは実現できないし、XPath に頼るか、自分でゴリゴリ DOM をトラバースしないといけない。

その点、Accessible Name がセレクタとして使える、というのはかなり嬉しい。
