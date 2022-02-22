# GraphQL との向き合い方 2022 年版(仮) Agenda

## GraphQL って何？

Facebook が開発し（ry

もう 10 歳！
https://twitter.com/mxstbr/status/1493579989383630852

## GraphQL を導入するモチベーション・タイミング

### モチベーション

T.B.D

- フロントエンドの要望に合わせて、いちいち Server に微細な修正を入れる必要がない
  - e.g. A の画面を分割することになったから、対応するエンドポイントも 2 つに分ける
  - Schema に定義されている範囲であれば、どのようにデータを取得するかはクライアントが自由にやればよい

### BFF との比較

### タイミング

巨大なサービスの 一部の API / 画面を GraphQL で置き換えるよりも、新規のサービスをなどに適用する方がメリットが大きい。

codegen など、必要なツールチェインを整えるのに一定の投資が必要。
「全体が GraphQL を使っている」状態にしておかないと、投資に見合ったリターンが得にくい。

## RESTish API と違うところ・似ているところ

### RESTish API と実は大差ないところ

結局 HTTP に載るという現実。

アプリケーションを運用する上で、REST friendly な基盤とどう統合するか

e.g. APM, SLO

(IMO) HTTP Status Code はインフラの様相を指し示すだけのものとして扱っておく。

逆に URL に通信内容を一定表現するよう、GraphQL 側から歩み寄っておくのもアリ。
例として、URL のクエリストリングに Opearion Name (Query や Mutaion の名前) を付与しておくだけで、Dev / Ops のやりやすさがぐっと向上する。

### RESTish API と違うところ

#### Supply Driven v.s. Demand Driven

Dataset の決定主体が Client Side にあるということ。

Server Side が知っているのは Schema(= 全部) に対して、その部分集合を決定する責務は Client Side(= 部分)が持つ。

さらに、Client の中でも、1 つの Query(= 全部) は枝葉末節の画面部品たる(= 部分) を意識しない。

Fragment Colocation / Query Composition

この思想に一番忠実な Framework が Facebook Relay.

最初期から、Higher Order Component たる `FragmentContainer` に始まり、現在の `useLazyFragment` に至るまで思想が一貫している。

### Tooling

codegen, validate, auto-complete

### Misc

- Cache
- Error Schema
- Persisted Queries

## 今後・気になる新機能

### Federation

Microservices /w GraphQL

Schema Stitching

Schema Directive Introspection

### Apollo Client の今後

Apollo Client v4 , Suspense 対応, `useFragment` など
