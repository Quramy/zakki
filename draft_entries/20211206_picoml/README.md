# 自作プログラミング言語とコンパイラの話

## はじめに

このエントリでは、僕が趣味で作っている PicoML という自作プログラミング言語の話を書こうと思う。

作ってから半年くらい経っているのだけれど、如何せん自作言語という無用の長物であるため、登壇などでコイツの話をすることもないため、ある種の供養みたいなものだと思ってもらって構わない。

## PicoML の概要

まずは実装した言語の紹介から。ざっくり以下の特徴を備えた言語である。

- ML ベースの自作プログラミング言語
- TypeScript で実装
- Node.js 上での即時評価機能と（REPL）と WebAssembly をターゲットにしたコンパイラの双方を提供

自分の勉強用の目的が強いため、REPL やコンパイラの CLI を動作させるのに一切の依存パッケージを必要としないようにしている。トークナイザや.wasm バイナリファイルの出力に至るまですべてフルスクラッチで記述した。気づいたら 13,000 行程度（本体 10,000 行/テストコード 3,000 行くらい）に育っている。

言語としての書き味は以下のようなイメージ。

```ocaml
let x = 20 in
let y = 10 in
x + y (* 30 *)
```

もうちょっと込み入った例として、1 から 10 までの整数の階乗の計算式を書くとこんな感じ。

```ocaml
let rec map = fun f -> fun list -> match list with [] -> [] | x::y -> (f x)::(map f y) in

let rec range = fun s -> fun e -> if s > e then [] else s::(range (s + 1) e) in

let rec fact = fun n -> if n < 2 then 1 else n * fact(n - 1) in

map fact (range 1 10)
```

`npm -g install pico-ml` から REPL とコンパイラをインストールできるようにしてあるが、Web Playground も用意している。Web Playground 版は、ブラウザの JavaScript で PicoML のコンパイラを動的に動かして WASM を生成し、それを実行するようにしている。画面の左下に表示されているのが、コンパイルされた WASM のバイトコードだ。

![Playground Screenshot](images/playground_capture.png)

[Web Playground](bfCYASwGdJTEkxYALS2yAdyrDYG0AusTKCIAHwgAPAFzSAnsIgAKLFICUspcjSq5aiFTwAoI6Egx4UJHgDmIDOQI1FuAvYZVVzsvd4gCoiDANPY0mla29krOANQQAIz6IPqGJmbQcNhIsJCYrhAEHqoEADwQAEwQfgTxEEEhBRAAVFk5SoUJKcZG2q2QShF2CQkADGpGSi3o6GSGkNR00hB88QA0FesAbOvlACzr8eUj6wDsR) からどうぞ。

上記の例のとおり、if 式や再帰関数定義、cons リストとパターンマッチなどをできるようにしている。基本的に OCaml のサブセットとなっている。

逆にできないことは山のようにある。

- 単一の式しか評価・コンパイルできない
- 扱えるデータ構造は整数・浮動小数・真偽値・関数・リストのみ。タプルとかない
- 標準関数のようなものは何も提供していない。`List.map` とかも自分で書く必要がある

## PicoML を作った理由

全体で 13,000 loc というと、僕にとってはまぁまぁなボリュームだ。

どれくらいのボリューム感なのか、他と比較してみると、例えば [TypeScript で GraphQL Client を便利に開発するためのツールを作っている話](https://quramy.medium.com/typescript-%E3%81%A7-graphql-client-%E3%82%92%E4%BE%BF%E5%88%A9%E3%81%AB%E9%96%8B%E7%99%BA%E3%81%99%E3%82%8B%E3%81%9F%E3%82%81%E3%81%AE%E3%83%84%E3%83%BC%E3%83%AB%E3%82%92%E4%BD%9C%E3%81%A3%E3%81%A6%E3%81%84%E3%82%8B%E8%A9%B1-b66dd2fc1579?p=b66dd2fc1579) で紹介した ts-graphql-plugin という、やはり TypeScript 製のツールがある。ts-graphql-plugin は、僕が真面目に作っている OSS 類としてはコードが多い部類なんだけど、試しに先程測ってみたら 9,000 loc 程度だったので、PicoML はその 1.5 倍近くあることになる。

そこまでの手間をかけてまで何がしたかったのか、と問われたとしても明確な答えが返せなくて「一回くらいはプログラミング言語を実装する」という経験を積んでおきたかったから、と言うしかない。

日頃、Web フロントエンド関連の仕事をしている中で、抽象構文木を扱ってちょっとしたツールのようなものを用意することは割と多いのだけれど、業務の中では全体のツールチェインの隙間を埋めるためのグルーコードになる。最初から最後まで自分だけで実装するという機会は、自分で作らない限り巡り会えないと考えていたからかもしれない。

## 製作時系列

およそ 2 ヶ月半くらいで一通りを作っている。その後はたまに機能を足したりしていた。

- Phase 1. 2021/03/08 - 2021/04/03: 構文解析, 評価, 型推論部分
- Phase 2. 2021/04/03 - 2021/05/07: WebAssembly へのコンパイラ
- Phase 3. 2021/05/12 - 2021/05/21: Web Playground UI

以前に [「プログラミング言語の基礎概念」を読んだ](https://quramy.medium.com/%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0%E8%A8%80%E8%AA%9E%E3%81%AE%E5%9F%BA%E7%A4%8E%E6%A6%82%E5%BF%B5-%E3%82%92%E8%AA%AD%E3%82%93%E3%81%A0-9d9d568968f0)
というエントリを書いたのだけれど、これが上記の Phase 1 に相当する部分。

その時点のコードベースから fork して、演算子などを追加しつつコンパイラ部分を追加したのが Phase 2 だ。この時期は最初の 10 日間ほどは実装は一切せずに、WebAssembly の勉強をしていた。以下は当時の勉強メモ。

![WebAssemblyを勉強している様子](images/wasm_study_log.png)

Phase 3 の Web Playground については、おまけみたいなものだ。CLI 側は気合入れて依存パッケージゼロで組んだけど、Playground 側は普通に React / Ace Editor でサクッと終わらせた。どうせ webpack でバンドルしてしまうので、zero dependencies の意味もフロントだとボヤけるし。
