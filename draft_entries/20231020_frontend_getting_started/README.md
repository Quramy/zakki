# Web フロントエンドの標準化でやっていることメモ

## 全体的なアーキテクチャの選定

フロントエンドエンジニアだけではなく、バックエンドエンジニアや SRE の意見も加味しつつ決める部分

- Next.js (App Router / Pages Router) どうするの？の検討や合意形成
  - SSR (Server Side Node.js) の有無
    - 最近はそこまで気にされなくなったとはいえ、フロントエンドがサーバー運用することにネガティブを感じる現場もまだある印象
- フロントエンド - バックエンド間の通信方式
  - OpenAPI なのか GraphQL なのか、と言った辺り
  - 上記の SSR 有無とも関連するが、ブラウザから直接 バックエンド API を呼び出すのか、インターネットフェイシングな BFF を用意してから、同一 VPC (または k8s namespace)内の通信とするのか
- 基盤構成のキャッチアップ
  - 普段は基本的に AWS 前提 (稀に GCP)
  - 生で EC2 使うのか、Docker 使うとして Fargate なのか EKS なのか、など
  - CDN はどこにいるのか、ロードバランサ (ALB 相当) はどこにいるのか、ブラウザから API サーバーに到達するまでの経路をイメージできるようにしておく

同じ様なコードを書くにせよ nginx に .js や .css を載せるのか、 Node.js のサーバーを Docker Image として用意して Deploy するのかで後続で気にすることは大きく異なる

## ソースコードの書き方を考える

これを済ませておかないと、「複数人で開発を開始するのが難しい」という系統に絞って考える。

質が悪いのは、ずーっとライブラリの選定だったり、lint のルールを議論し続けて何も開発が進まないような状況に陥ってしまうこと。
全部を決めきる、準備しきるのは不可能なので、作りたい画面の特性に合わせて取捨選択をすること。
(e.g. 回遊閲覧がメインのサイトであれば、Form や更新系処理の検討は後回しにする)

- ライブラリの選定, インストール
  - お約束系:
    - TypeScript, prettier, ESLint あたり
      - この段階で lint のルールについてごちゃごちゃ言っても時間の無駄間があるので、 standard 系の軽めのやつを入れておしまいにしておく
      - 場合によっては Stylelint とかも
  - CSS の書き方を決める
    - CSS in JS or CSS Modules なのか、tailwind を使うのか、のような部分
      - Zero Runtime 系の場合、Meta Framework が用意している Build 周りと親和するかも考慮点
    - 非 atomic css の場合、デザイントークンの管理方法も考えるべき
  - Component UI ライブラリ (Material UI とかそういう系)
    - ちゃんとデザイナーが Sketch や Figma で VD を用意してくれているのであれば、自前で CSS を書いてしまえばよいと思っている
  - State Management
    - Redux とかそういう部分
      - とはいえ、2023 年にもなって Redux を新規に選定することはほぼないはず
    - Atomic State Management なライブラリを検討する方がよい
      - Recoil は非アクティブなため、リスク高い
  - Form ライブラリ
    - React の場合、react-hook-form 一強？
    - RHF だけでなく、resolver (e.g. zod) も検討しておく
  - API 通信方式
    - ここで言っているのは、ブラウザ - Node.js 間の通信ではなく、フロントエンド - バックエンド通信の意。
    - OpenAPI の場合
      - yaml をフロントエンドが書くのか、バックエンドが書くのかによって大きく異なる
      - バックエンドが API 定義を書く場合、openapi-genenerator の設定を行う
      - フロントエンドが API 定義を書く場合、YAML を生成するための TypeScript と親和するツールを用意する(e.g. zodios)
    - GraphQL の場合
      - ランタイムの選定: Apollo Client なのか urql なのか
      - SDL を引っ張ってきて、TypeScript Client 生成を行えるようにする (e.g. graphql-codegen)
    - スタブサーバー
      - バックエンド API を叩くのが面倒、並行開発なのでまだ存在しない、のような場合に検討する
        - そもそもローカルで普通にバックエンド API を動作させた方が楽なのであれば、そっちを使えばよい
      - graphql-mock や MSW を BFF に仕込む、など
  - Component カタログ
    - 実質 Storybook 以外の選択肢が無い
  - テスト関係
    - 最低限の Component Unit Test が書けるくらいのセットアップを事前に済ませておくと楽
    - jest, jest-environment-jsdom, testing-lib, のインストール及び、global な stub の設定など(e.g. Next Router)
- ディレクトリ構成
  - 基本的には選択した Meta Framework の構成に従うようにする
  - 規模が大きめの場合、NPM workspaces の利用も検討する
    - ビルドが難しくなりがちなので、trade off も考慮する
  - Atomic Design を採用しているのであれば、共通コンポーネントディレクトリをどう切るのか、などを考える
  - `utils` のような名前のディレクトリを切ってしまうと将来のゴミ箱になるので、名前はちゃんとかんがえるべき
  - 先に決めておくと楽な箇所
    - ただの関数群: e.g. `src/functions`
    - ライブラリ用グルーコード. Storybook の Decorator や、独自 Redux Middleware: e.g. `src/support/storybook`, `src/support/redux`
- Scaffold
  - hygen や schematics など
  - Component
    - CSS, Storybook, test に何を使うか、及びディレクトリ構成の大まかな検討が完了していれば、Component を自動生成するようにしておく
  - API Client
    - 自分で組むことは無いので、Open API Generator や GraphQL Codegen などの設定と同じ意味
- PoC の作成. 各種ライブラリのセットアップと並行して、各アプリケーション処理方式が想定通り成り立っているかを試していく. いわゆるサンプル実装.
  - API 通信について、正常系だけでなく、非正常系のハンドリング方法も考えておくこと
    - Loading Status はどこから取得するのか, Suspense 使うのか, ...
    - エラーハンドリング は後回しになりがちだが、後から詳細な要件が出てきても改修しやすい仕組み(共通的なエラーハンドリングインターセプタ系統）を噛ませられるようになっているかどうか, ...
  - (SSR を採用している場合)
    - Client Site Routing と Server Side
    - API やその結果の Client State の初期値がどう hydrate されるのかを検証・理解しておく

## BFF 周りの基盤選定

BFF(Backend For Frontend) をフロントエンドで開発・運用する場合に考えること

- 環境変数の取り扱い
  - Server Side に留めるもの (各種 Secret や Backend Service の URL)と、ブラウザまで露出させるべきもの (インターネットから見たサービスの FQDN など) を分別して管理できるようにしておく
- セッション関連
  - 大概が認可とセットになる
  - Redis / DynamoDB などの middleware と結合する箇所の設定
    - express-session や next-session など
- ロギング
  - ローカル開発時以外は構造化ログで出力しておく、など実行環境に合わせてロギングミドルウェア(pino や morgan) の設定を仕込む

## CSS 関連

Prj が走り出してしまうと手を出しにくい部分は以下.

- 共通変数の定義
  - そもそも「どこまで共通変数化するのか」を定める
    - Figma や Sketch の VD が存在している場合、VD に記載されていないものに対して無闇に変数化しても、作業効率が下がるだけ
    - e.g. Figma のカラーパレットに存在するもののみ、Custom Variables に定義するようにする
  - 実装形態は CSS in JS or CSS Modules or Tailwind or ... で異なるが、考え方そのものはあまり変わらないはず
- リセット・ノーマライズ
  - いわゆる UA の標準スタイルを打ち消す系
  - `box-sizing` は後から変更するのが面倒な系統なので、真っ先に対応しておくべき
- ブレークポイントと優先順位
  - レスポンシブ対応が必要な場合、SP First なのか、Laptop First なのかを決めておく
  - 共通定数として保存しておくと楽
    - CSS Modules の場合、 Custom Media 用の PostCSS Plugin を設定しておく
- rem v.s. px どっちを決める
  - 長さに限らず、標準で用いる単位は ある程度 Stylelint で縛っておくとよい

## ビルドパイプライン

レポジトリから実行環境に Deploy するまでの自動化部分。
すべてをフロントエンドがやる訳では無いが、インフラ・SRE が全部やってくれるわけではないので、分担しつつ恊働していく。

T.B.D.

## エラーハンドリングの設計

T.B.D.

## カスタムイベント計測

T.B.D.

## 計装

T.B.D.
