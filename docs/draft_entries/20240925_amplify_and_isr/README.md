# AWS Amplify と Next.js ISR についての覚書

今まで食わず嫌い気味に避けてきた Amplify を少しだけ触ってみたので、その覚書。

一口に Amplify といっても色々なコンポーネントがあるが、ここで取り上げるのは主に Amplify Hosting について。

結論を先に書いておくと、Incremental Static Regeneration(ISR) メインで Next.js を使いたい場合に、ホスティング先に Amplify を選択するのは避けたほうがよい。

https://docs.aws.amazon.com/amplify/latest/userguide/ssr-Amplifysupport.html#supported-unsupported-features に記載されているとおり、 Amplify では Incremental Static Regeneration(ISR) がサポートされている。

## コールドスタートが遅い

Amplify Hosting Compute の裏側では Lambda が動作している。

Lambda ではよく知られた問題だと思うが、コールドスタートが発生するとかなり遅い。

ランタイムに何を選択するかで変わってくるとは思うが、 AWS のレポジトリで公表されていた諸元がある。
Node.js ランタイムの場合、コールドスタートがかかった際には Lambda の実行時間は 1,500 ミリ秒を超える（測定している関数は DynamoDB に 1 レコード Put するだけの極めてシンプルな関数）。
一方、ウォームスタートであれば、100 ミリ秒以下である。

ref: https://github.com/awslabs/llrt

ところで Next.js の ISR は、Stale while revalidate ライクな挙動を取る。

- Cache が fresh であれば、キャッシュしてあるコンテンツをレスポンスとして返す
- Cache が stale していれば、stale したキャッシュをレスポンスとして返す
  - と同時に、キャッシュを revalidate する
- Cache が存在しなければ、キャッシュを生成した後にレスポンスを返す

Next.js 、すなわち Lambda がキャッシュの状態判定を行うわけだが、コールドスタートの場合、Next.js の起動を待って初めて上記のキャッシュ状態判定に到達することになるため、エンドユーザーから見た TTFB はかなり悪くなる。

これが通常の Lambda であれば、Provisioned Concurrency を適切に設定すればコールドスタートの頻度をチューニング可能である。

ref: https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html

しかし Amplify の場合、Amplify が作成した Lambda を開発者が触ることができないため(Amplify CLI で `amplify add function` のように追加した Lambda 関数ではなく、 Amplify Hosting Compute の裏で動いている Lambda のことである)、チューニングも厳しい。

ちなみに、自分のアカウント上に Lambda のリソースが作成されていないのに、なぜ Lambda であると言い切っているのかというと、以下を Amplify Hosting Compute で Deploy し、`AWS_Lambda_nodejs20.x` の値を確認したため。

```tsx
/* src/app/page.tsx */

export const dynamic = "force-dynamic";

export default function Page() {
  const runtime = process.env.AWS_EXECUTION_ENV;
  return <>${runtime}</>;
}
```

## コールドスタート時にページが巻き戻る

コールドスタートのパフォーマンス上の問題に目を瞑ったとして、まだ問題がある。

Lambda によって起動した Next.js のプロセスがキャッシュ有無を判定する、と書いた。

> - Cache が fresh であれば、キャッシュしてあるコンテンツをレスポンスとして返す

この Cache は Next.js の初期設定そのままのナイーブなファイルキャッシュであり、Lambda 実行環境間で共有されている訳ではない。

`next build` の過程で生成された直後の `.next/server` 以下が Lambda のエフェメラルストレージ上に展開されているだけの状態になる。
すなわち、コールドスタートに割りあたったユーザーは、1,000 ミリ秒以上待たされた挙げ句に、ビルド時点でのコンテンツが返却されることになる。

これを回避するには、Lambda 実行環境の外部でキャッシュを保持できるように Next.js のカスタムキャッシュハンドラを自分で作成するしかない。

https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath

Amplify で作成可能なリソースとなると DynamoDB くらいしか思い浮かばない。

## Full Route Cache の書き込みに失敗する

Next.js のキャッシュにまつわる問題はまだ存在している。

Next.js では、ルーティング方法に App Router / Pages Router の 2 通りが存在することは有名と思うが、App Router を選択している場合、新しくキャッシュを書き込めない。以下のように `EROFS` 例外が発生する。

```
Failed to update prerender cache for /index [Error: EROFS: read-only file system, open '/tmp/app/.next/server/app/index.html']
```

ref: https://github.com/aws-amplify/amplify-hosting/issues/3707#issuecomment-2066667185

この Issue については自分でも少し調べたのだが、どうも Lambda の実行環境において、App Router 用のキャッシュ領域を正しく扱えていないように思える (Pages Router が書き込む領域のみを特別扱いしている節がある)。

## On-Demand Revalidate できない

コールドスタートを踏むと応答が悪く、また Next.js が持つキャッシュ機構と相性が悪いのであれば、Revalidate されるまでの期間を大きく取って、CloudFront のキャッシュヒットレートを高くするという戦略が考えられる。
この場合、ページの元となるコンテンツを更新した時に明示的にキャッシュを Revalidate させる、いわゆる On-demand Revalidate を併用する必要がある。

が、これについては Amplify が公式にサポートしていない旨を書いている。

上で書いたように、DynamoDB でカスタムキャッシュハンドラを用意すれば、Next.js の持つキャッシュは外から制御可能になるだろうが、さらに CloudFront の API も実行しないといけない。
そして、Lambda が隠蔽されているのと同じく、CloudFront も Amplify に隠蔽されているため、これをやりたければ、Amplify の CloudFront よりも手前に、自前で CloudFront を置いてやる必要がありそう。

## Static Site Generation... ?

要件上問題にならないのであれば、特定の規模までであれば SSG の方がシンプルに作れるのだが、、Next.js v14 以降のアプリケーションを SSG アプリとして Deploy するのも難しい。

Amplify Hosting が Static / Compute を見分ける方法が「Package json の build script に `next export` と記述されているかどうか」となっているためである。

```json
{
  "scripts": {
    "build": "next export"
  }
}
```

ref: https://github.com/aws-amplify/amplify-hosting/issues/3872

Next.js から `export` コマンドが削除されたのは、v14.0.0 のタイミング、すなわち 2023 年の 10 月で、上記の GitHub Issue もその直後には Open されているのに、そのままなのはどうなのか。もっというと、App Router における Static Export の方法として、`next.config.js` で行うやり方そのものは 2023 年の 4 月から案内されていた。

ref: https://nextjs.org/blog/next-13-3#static-export-for-app-router

## おわりに

確かに、Amplify CLI や Amplify コンソールから簡単にデプロイできるし、AWS の知識もほぼ不要だし、無料枠もそこそこあるため、比較的小規模なプロジェクトを短時間で立ち上げるのには向いていると思っていた。
しかしながら、触ってみた感想としては最悪に近い。自分では絶対に運用したくない。

コールドスタートが遅い件に関しては LLRT に期待して目をつぶったとしても、キャッシュが共有されていない・書き込めない、Static Export もできないという体たらくでは、とてもじゃないが「App Router に対応している」とは言えないのではなかろうか。
