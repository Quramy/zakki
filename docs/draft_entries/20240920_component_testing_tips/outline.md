## Title

- 上手に向き合うコンポーネントテスト

## Description

フロントエンドでも自動テストを書くのは当たり前となりつつある昨今。コンポーネントテストを上手に運用する Tips を紹介します。

## Outline

- 導入編
  - なんのための自動テスト？
    - 質がスピードに寄与する
  - 最初からやる？
    - ド新規で VRT やる？ -> やらない
      - 日々見た目が変わるを当たり前としている中で、毎度確認するか？と言われたら工数かけてまでやる必要ある？と思う
      - じゃぁいついれんの？
        - 初回リリースなどのイベント前後
  - VRT や Component Test を運用できるように備えておくことは重要
    - Visual Testing はただの Assertion 手段
    - 無からテストケースは生まれない
  - Component Test の位置付け
    - 静的解析
    - 単体テスト/Component テスト/E2E
  - Component Test に Storybook を用いている理由
    - ~~正直、昔ほどには Storybook が好きではない~~ (これは口頭での補足に留める)
    - 「複数のテスト手法をバランス良く」となった場合に、Component テストで利用する基盤はなるべく揃えておきたい
      - Jest / Vitest から単体テストとして再利用できる
      - Component カタログとして利用できる
      - Visual Testing に利用できる
      - Interaction Testing に利用できる
    - Storybook 一強
      - Alternative Storybook が育たない...
- 実践編
  - 1. 殺せ VRT Flakiness
    - False Positive は最大の敵. 特に Visual Testing の場合はコイツを以下にのさばらせないかが成功の鍵
    - Threshold
      - 角丸などで影響受けることはある
        - 実数にして 10px x 10px 程度を許容しておくとよい
        - https://playwright.dev/docs/test-snapshots#maxdiffpixels
        - 100 程度で偽陰性になることは滅多にない
    - どうしようもないときは skip
      - Story 全体のスクショを取らない
      - 動画 Component や iframe など、自分たちで制御できない部分については、「VRT 時のみ、当該 Component を Placeholder に差し替える」をやることもある
  - 2. 削れ 実行時間
    - Push した後の CI, どれくらい我慢できますか？
      - 5 min ~ 10 min くらい？
    - a: 並列実行
      - CI のマシンパワーを増やして component テストの並列数を上げる
        - e.g. `--max-workers`
      - CI の実行環境ごと並列化する
        - e.g. `--shard`
    - b: ランナーを分ける
      - Component テストの全てにブラウザが必要というわけではないはず
        - jsdom で十分なのであれば、Jest / Vitest の テストコード側に移す
    - c: 無意味な `setTimeout` をやめろ
  - 3. CSF を使いこなせ
    - `storybook/test`, Storybook App
    - CSF でできることを押さえておくと、コンポーネントテストの記述の幅が広がる
    - CSF の関係
      - Loaders -> (Story) の起動 -> Decorator (Story) -> Play (ここは図にする)

<!--
## Outline

- 導入編
  - なんのための自動テスト？
    - 質がスピードに寄与する
  - ド新規で VRT やる？ -> やらない
    - 日々見た目が変わるを当たり前としている中で、毎度確認するか？と言われたら工数かけてまでやる必要ある？と思う
    - じゃぁいついれんの？
      - 初回リリースなどのイベント前後
  - VRT や Component Test を運用できるように備えておくことは重要
  - Component Test に Storybook を用いている理由
    - 「複数のテスト手法をバランス良く」となった場合に、Component テストで利用する基盤はなるべく揃えておきたい
      - 静的解析/単体テスト/Component テスト/E2E
        - Jest / Vitest から単体テストとして再利用できる
        - Component カタログとして利用できる
        - Visual Testing に利用できる
      - Interaction Testing に利用できる
    - 正直、昔ほどには Storybook が好きではない
- 実践編
  - 主に Storybook のノウハウを並べていきます
  - CSF を使いこなす
    - arguments
    - play
      - 個人的には、「重要なシナリオ」を E2E (Playright) で実装し、ただのインタラクション(ボタンを押したら xxx される) 程度は Play fn で良いと思っている
    - decorator
      - Higher Order Function
      - コンテキストの差し替え
    - loaders
      - Story の事前処理を定義できる
      - Fetch then render / Render as you fetch のパターンに向いている
        - 図
  - 番外編
    - Module Mocking
      - jest や vitest, Node.js test runner における Module Mocking とは大分毛色が違う
      - Subpath import を interface としているが、これは流行るんだろうか...?
-->

<!--
## Outline

- 導入するときのコツ:
  - テスト手段を複数用意しておくこと
  - 向き不向きがある
    - Jest or Vitest: 早い, Component Test 普通, E2E 遅い
    - どれか一つだけとしていると後で困る (それぞれ薄く入れておくとよい)
- Component Testing
- No test
- Fragile test
  - Snapshot に頼りすぎ
- Flaky test
  - VRT での flakiness
    - 角丸などが影響受けることはある
      - 実数にして 10px x 10px 程度を許容しておくとよい
      - https://playwright.dev/docs/test-snapshots#maxdiffpixels
      - 100 程度で偽陰性になることは滅多にない
  - Flakiness は最大の敵:
- Slow test
  - 自動テストが遅くてうんざり
    - 待てる時間の限度ってどのくらい？
      - 観測範囲では 5 ~ 10 minutes 辺りにしきい値がありそう
  - 削れるところは削れ
    - 「よくわからないから wait」しないで
- Obscure
  - 何がしたいのか分からない、よむのが辛い
- Bad smell
-->
