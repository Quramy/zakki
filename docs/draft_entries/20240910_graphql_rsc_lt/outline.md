# Title

GraphQL あるいは React における自律的なデータ取得について

# Outlines

- フロントエンドから見た GraphQL の利点
  - パフォーマンスとメンテナンシビリティの両立
    - GraphQL がない場合、UX / DX のトレードオフになってしまう
      - Root data fetch: Good perf, Bad DX
      - Leaf data fetch: Bad perf, Good DX
  - GraphQL の哲学
    - Client / Server ともに Leaf に処理を記述する
    - 言い換えると、各モジュールが自律的. Fat Controller や Fat Component が生まれにくい
      - Client: Leaf で宣言(fragment), Root(Query) はまとめ上げるだけ
      - Server: フィールドリゾルバは「自分自身が何を返すか」だけを意識すればよい
    - Leaf <-> Root のやりとりはフレームワークに任せる
    - (図解): Fragment Colocation + Resolver の様子の一枚絵
- GraphQL 代替としての React Server Component
  - RSC(React Server Components) とは
    - **サーバーでのみ** 動作する React コンポーネントが爆誕
      - 従来の Server Side Rendering は ブラウザで動作する React コンポーネントを サーバーでも動作させているだけ
  - SC では "Leaf data fetch" が Performance 上のトレードオフではなくなる
    - Server でしかできないこと、すなわち データ解決非同期IO
  - Fragment Container + Resolver = Async Server Component
    - GraphQL Resolver が Type Data を返す代わりに、Server Component が「Data + 描画された UI の断片」を返す
    - RSC の場合、「どこまで Server で Render しておくか」の境界には選択の余地がある (Client Boundary)
  - この構成で開発するために必要なこと
    - 対応フレームワーク: 2024.09 現在は実質的に Next.js のみ. Remix(React Router) は対応を表明している
    - GraphQL Resolver 開発の知見
      - e.g. N + 1 問題への対応パターン, Lazy Loading
      - GraphQL Sever Side 経験者はすんなり RSC を始められると思う
      - 従来の Web MVC な開発の感覚の方が RSC をとっつきづらく感じさせるはず
- GraphQL の今後
  - [IMO] RSC + GraphQL はやりたくない
    - 混ぜるなキケン
      - 同じことが実現できる大きく異なる方法を混在させることに対する忌避感
      - 重複しているものが多すぎる (根本にある解決しようとした問題が同じなのだから当然)
        - Fragment colocation v.s. Fetching data on Leaf
        - Apollo や Relay の Store 層 v.s. Next.js の Cache
        - `@defer` v.s. React Streaming SSR
    - RSC を使わずに、GraphQL + Relay(or Apollo Client) + Remix SPA mode の構成の方がまだマシ
  - LSUDs API Gateway としての GraphQL は残っていくと思う
    - 例: Shopify のようなパターン
  - 「現行資産としての GraphQL サーバー + 新規のフロントエンドに RSC」への折り合いは、ユーザーランドにとって今後の課題
    - そもそも RSC が膾炙しない可能性もありうるけど. Next.js 以外でも使えるようになってからが本番？
    - 動かすだけなら別に簡単. ちゃんとやるのが難しいはず
      - e.g. Fragment colocation する・しないをルール決めする
    - Relay は RSC 統合を一度断念している
    - Apollo のやつはよくわからん(とりあえず動くだけ. 思想を感じない代物)

<!--

- GraphQL を RSC で置換できるパターン
  - クライアントが Web ブラウザのみであること
  - RSC を利用できること
    - 2024 現在は実質的に Next.js のみ.
    - react-router(Remix) v7 が RSC に対応したら裾野が広がるかも？
  - 例:
    - T3 Stack(= React + tRPC + Prisma ORM)はよりシンプルに、コンポーネント自身が Prisma Client を呼び出す構成とできる
    - Gatsby における開発体験がそのまま Next.js の SSG(ISR) でも利用可能に
-->
