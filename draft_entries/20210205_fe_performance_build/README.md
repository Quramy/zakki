# Front-End Performance Checklist 2021 の Build Optimization を読む

https://www.smashingmagazine.com/2021/01/front-end-performance-2021-free-pdf-checklist/#build-optimizations

## 28. Have we defined our priorities?

ページが必要とするアセットを列挙して、それぞれのアセットがどのグループに属するかを洗い出すべし。

- _core_ : レガシーブラウザでも動作させる必要のあるコアコンテンツ。全員がアクセス可能である。
- _enhanced_ : よりリッチな体験。ブラウザ毎に異なってもよい部分。
- _extra_: 必須ではないコンテンツ。web font など、遅延読み込み可能なもの

[過去に Smashing Magazine 自体が行ったパフォーマンス改善](https://www.smashingmagazine.com/2014/09/improving-smashing-magazine-performance-case-study/) も参考のこと。

core > enhanced > extra の順で最適化をする。

## 29. Do you use native JavaScript modules in production?

core 向けと enhanced 向けに提供モジュールを分ける。

テクニックとしては、`<script type="module">` を使った Differential Serviing パターンを使う。

core(レガシーブラウザ): トランスパイル済み, polyfill 込
enhanced(モダンブラウザ): トランスパイル無し（もしくは downlevel が小さい）, polyfill 無

ただし、一部のブラウザではこの hack が逆効果に働くことがある。 https://gist.github.com/jakub-g/5fc11af85a061ca29cc84892f1059fec

![](./differntial_loading_caveat.png)

(Chromium Edge, iOS Safari 14 にパッチがあたったので、double fetch 問題は発生しなさそう)

`module/nomodule` の feature detection だけでは足りないことがある。Chrome が搭載されている安価な Android 端末などに、enhanced な機能が必要か？という話。

将来的に、Client Hint Header でデバイスメモリを見つつ、これも serve する js ファイルの判断基準に加えたらよいのでは。

## 30. Are you using tree-shaking, scope hoisting and code-splitting?

- Tree Shaking: Code Elimination のテクニック
- Scope Hoisting: ある種の inline 展開でコード量を削減するテクニック

Note: webpack 4 時点ではどちらも on だし、あんまり気にする必要ない

### Code Splitting

SPA の場合、bootstrap のスピードを上げることは重要。FW 毎に色々なテクニックがあるから、それを参照すべし。

Note: Angular のテクニックとして紹介されているのは、https://github.com/mgechev/angular-performance-checklist/blob/master/README.ja-JP.md#%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E3%81%AE%E9%81%85%E5%BB%B6%E8%AA%AD%E3%81%BF%E8%BE%BC%E3%81%BF のあたりなど。

## 31. Can we improve Webpack's output?

webpack の設定をもっと頑張ることで、バンドルサイズをより削減できるチャンスがあるよ、という話。

### Tree Shaking と Side Effects の話

Tree Shaking で `import` / `export` は削られるが、関数呼び出しのコード

Note: [webpack 側の説明](https://webpack.js.org/guides/tree-shaking/) をちゃんと読むのが一番わかりやすい。

### 雑多な plugin 達の紹介

他にも、状況に応じて、bundle サイズを削減できるような plugin が色々あるので、適用可能かどうかを検討すべし。

- https://www.npmjs.com/package/purgecss-webpack-plugin
- https://www.npmjs.com/package/google-fonts-webpack-plugin
- etc...

## 32. Can you offload JavaScript into a Web Worker?

UI Main Thread から Web Worker に処理を逃がすことで、CPU 詰まりを解消できるよ、という話。

典型的なユースケースとして、PWA / SPA で、後で利用するデータを prefetch しておくような処理は Web Worker に逃がすことができる。

https://github.com/GoogleChromeLabs/comlink のようなフレームワークを使えば、Thread 間のメッセージングを比較的容易に書ける。

v80 以降の Chrome だと、worker の js ファイルを module 読み込みできる。Main Thread での `import()` を利用するのと同じような遅延ロードが可能、ということ。

```js
const worker = new Worker(url, { type: "module" });
```

Note:
「実際にどれくらい効果があるの？」的な話につながるリソースは見当たらず。 [CDS 2019 の PROXX](https://web.dev/off-main-thread/) が 参考として挙げられていたけど、アレを一般的なユースケースというには疑問符が付くところ。

一般的に、Worker 化に向くか向かないかとして、書きの基準がある。

- Worker に向いている処理: 実行に時間がかかるコードブロック, メッセージの入出力データ量が小さい, request/response モデルに従っている
- Worker に向いていない処理: DOM に依存する, 遅延が許されない処理,

Note: そもそも Worker からは DOM 扱えないからそりゃそうだろ、って気がする

## 33. Can you offload "hot paths" to WebAssembly?

WebAssembly は JS を置き換えるためのものではない。

大概の Web アプリケーションでは JavaScript の方が向いているが、WASM は"computationally intensive web apps", すなわち計算コストそのものが効いてくる（e.g. ゲームなど）は WASM での実装が適している。

ほぼすべてのブラウザで WASM を動かすことができる、また JS -> WASM 呼び出しのオーバーヘッドも改善されている。

Note: この節には一般的なことしか書いてなかったので、[Front-End Study #2 の chikoski さん動画](https://youtu.be/Ga8P_buwXnw?t=3951) とか観た方がよいかな。

## 34. Do we serve legacy code only to legacy browsers?

モダンブラウザはすべて ESM サポートされているので、これら向けには Transpile ターゲットを ES2017 にできる。

（IE などの）レガシーブラウザには、 `<script nomodule>` 従来の bundle を提供すればよい。

https://web.dev/publish-modern-javascript/ にモダン・レガシーな JavaScript をどのようにトランスパイルすべきかのガイドラインがあるので、参考にするとよい。

https://estimator.dev/ に自分のドメインを入力すれば、どれくらいの削減効果があるのかを事前に評価できる。

Note: "29. Do you use native JavaScript modules in production?" の節と言ってること一緒なような。。。

## 35. Identify and rewrite legacy code with incremental decoupling.

長期に渡るサービスは、レガシーなコードが塵積になる傾向がある。
まずは、（それが大変であっても）レガシーコードのリファクタリングに必要な時間を評価しよう。

レガシーコードが呼び出されている割合を定期的にモニタリングし、それが上昇傾向に無いことを確認すべし。
レガシーなライブラリを呼び出すコードが PR されたら、警告するようにするとよい。

![](https://github.blog/wp-content/uploads/2018/09/jquery-usage.png)

## 36. Identify and remove unused CSS/JS.

## 37. Trim the size of your JavaScript bundles.

## 38. Do we use partial hydration?

## 39. Have we optimized the strategy for React/SPA?

## 40. Are you using predictive prefetching for JavaScript chunks?

## 41. Take advantage of optimizations for your target JavaScript engine.

## 42. Always prefer to self-host third-party assets.

## 43. Constrain the impact of third-party scripts.

## 44. Set HTTP cache headers properly.
