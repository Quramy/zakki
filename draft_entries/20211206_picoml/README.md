# 自作プログラミング言語と WebAssembly コンパイラ

## はじめに

このエントリでは、僕が趣味で作っている PicoML という自作プログラミング言語の話を書こうと思う。

作ってから半年くらい経っているのだけれど、如何せん自作言語という無用の長物であり、登壇などでコイツの話をすることもないため、ある種の供養みたいなものだと思ってもらって構わない。

## PicoML の概要

まずは実装した言語の紹介から。ざっくり以下の特徴を備えた言語である。

- ML ベースの文法による関数型言語
- 単純 let 多相型推論による型チェック
- CLI として、Node.js 上での即時評価器（REPL）と WebAssembly をターゲットにしたコンパイラの双方を提供

REPL やコンパイラの実装は TypeScript で行っている。自分の勉強が主目的であったため、REPL やコンパイラの CLI を動作させるのに一切の依存 npm パッケージを必要としないようにしている。トークナイザや.wasm バイナリファイルの出力に至るまですべてフルスクラッチで記述した。気づいたら 13,000 行程度（本体 10,000 行/テストコード 3,000 行くらい）に育っている。

言語としての書き味は以下のようなイメージ。

```ocaml
let x = 10 in
let y = 20 in
x + y
```

```ocaml
let twice = fun x -> x * 2 in
twice 10

(* 以下でも可 *)

let twice x = x * 2 in
twice 10
```

もうちょっと込み入った例として、1 から 10 までの整数の階乗の計算式を書くとこんな感じ。

```ocaml
let rec map f list = match list with [] -> [] | x::y -> (f x)::(map f y) in

let rec range s e = if s > e then [] else s::(range (s + 1) e) in

let rec fact n = if n < 2 then 1 else n * fact(n - 1) in

map fact (range 1 10)
```

`npm -g install pico-ml` から REPL とコンパイラをインストールできるようにしてあるが、Web Playground も用意している。Web Playground 版は、ブラウザの JavaScript で PicoML のコンパイラを動的に動かして WASM を生成し、それを実行するようにしている。画面の左下に表示されているのが、コンパイルされた WASM のバイトコードだ。

![Playground Screenshot](images/playground_capture.png)

[Web Playground](https://quramy.github.io/pico-ml/#code=DYUwLgBATiDGEFsCGAHCAzCwCWBnSAvIkmLABZZ6QDu2YFA2gLoQC0AfBMxAD4QAeALkEBPNpwAUmfgEphE5GkwiZEbADsAUJtCQY8KEnUBzEBFwQzRbJgucz9EOq4sQwXGdzzDJsxIsA1BAAjKogqhrautBwGEiwkM7WmM4APBAATBCOzsGW7mbOAFRxCRLOrCERWpqKpZASPqYhIQAMMkA) からどうぞ。

上述の例のとおり、if 式や再帰関数定義、cons リストとパターンマッチなどをできるようにしている。基本的に OCaml のサブセットとなっている。

逆にできないことは山のようにある。

- 扱えるデータ構造は整数・浮動小数・真偽値・関数・リストのみ。タプルとかない
- 評価・コンパイルできるのは式は一つまで。 utop のように「トップレベルに様々な変数を束縛して、後続の式から参照する」みたいなことはできない
- 標準関数のようなものは何も提供していない。`List.map` とかも自分で書く必要があるし、 `print` 関数もない
- etc,,,

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

## 得たもの

冒頭で「無用の長物」と書いたが、フルスクラッチで PicoML というプログラミング言語の実装をしてみたからこそ得たものもあったと思う。

- プログラムを書くことそのものの楽しさ
- 新しい言語の知識
- 既に持っていた知識の再確認

コンパイラを実装するというのを言い換えると「プログラムを書くプログラムを書く」という意味になるけど、そんなの楽しいに決まっている。

PicoML の場合、やっていることは「TypeScript で OCaml を WebAssembly に生成する」ことになる。実装前の段階での僕とって勝手知ったる言語は TypeScript だけだったので、WebAssembly や OCaml は仕様書や入門者向けのドキュメントを読み漁りながら 1 から身につけていく作業だった。

特に [WebAssembly Specifications](https://webassembly.github.io/spec/core/) は実装期間中は毎日のように読んでいたので、その過程で WAT (WebAssembly Text Format) の読み書きも体得できたし、Interface Types や Module Linking などの [WebAssembly Spec Proposal](https://github.com/WebAssembly/proposals) を追いかけるモチベーションにもつながった。

一方で、既に自分自身の中に溜まっていた経験を再度引っ張りだすような部分も少なからずあった。抽象構文木関連の処理という意味では実装したことのある系統の処理も結構あって、たとえば「TypeScript で GraphQL から TypeScript の型定義を生成する」という機能が [ts-graphql-plugin](https://github.com/Quramy/ts-graphql-plugin) にあるのだけど、これもある種のコンパイラだ。[DIY GraphQL Codegen](https://speakerdeck.com/quramy/diy-graphql-codegen?slide=21) という資料に AST to AST の概念をまとめたんだけど、この経験のお陰で PicoML だろうと（多分他の言語だろうと）エッセンスを抑えておけば、再利用のできる知識だと認識できた。

繰り返しになるのだけど、言語実装というのは本当に楽しくて、[低レイヤを知りたい人のための C コンパイラ作成入門](https://www.sigbus.info/compilerbook#%E3%81%AF%E3%81%98%E3%82%81%E3%81%AB) がその楽しさを説明してくれている。

> コンパイラ作成は大変楽しい作業です。最初のころはバカバカしいくらい単純なことしかできなかった自作言語が、開発を続けていくとたちまちのうちに自分でも驚くくらい C 言語っぽく成長していって、まるで魔法のようにうまく動くようになります。実際に開発をしてみると、その時点でうまくコンパイルできるとは思えない大きめのテストコードがエラーなしにコンパイルできて、完全に正しく動くことに驚くことがよくあります。そういうコードはコンパイル結果のアセンブリを見ても自分ではすぐには理解できません。時折、自作のコンパイラが作者である自分を超える知性を持っているように感じることすらあります。コンパイラは仕組みがわかっていても、どことなく、なぜここまでうまく動くのか不思議な感じがするプログラムです。きっとあなたもその魅力に夢中になることでしょう。

プログラマ万人がやるべき、とは思わない。前述の通り、そこそこ可処分時間が持っていかれるのは間違いないし、作ったところでその成果物自体が直接何かの役に立つことはまぁないだろう。でも、それを補って余りある楽しさがあるので、興味をもってくれる人がいたら、是非何かしらの言語実装にトライしてみてほしいと思う。

## プログラミング言語をつくる流れ

以降では、PicoML の中身を説明するにあたって「どのような流れで言語が実装されていくか」の概略を解説していきたいと思う。

- Design
- Abstract Syntax Tree
- Evaluate expression immediately
- Separate Frontend and Backend
- Compile to WASM Stack machine
- Error reporting via type system

1. 言語機能・仕様の設計
1. 抽象構文木の設計と実装
1. 即時評価機能の実装
1. コンパイル機能の実装
1. 型チェック機能の実装

まずは「1. 言語機能・仕様の設計」から。

ここで考えることは、本来は山のようにある。関数型なのか手続き型なのか、変数をどのように取り扱わせるのか、評価戦略は正格なのか遅延なのか、etc...
とはいえ、本当にゼロから設計することはなく、自分が知っている何かしらの言語を参考にすすめる筈だ。PicoML の場合、最初から「関数型言語であり、文法は OCaml のサブセット」という制約を強いているため、「OCaml の仕様のうち、どこまでを取り込むか」という意味になる。

また、設計したすべての機能を最初から作り込むのは無理がある。

ここでは一旦、「0 ~ 127 までの整数の足算と掛算ができる言語」というものを考える。これだって OCaml のサブセットであることには変わりない。

```ocaml
1 + 2
```

```ocaml
1 + 2 * 3
```

```ocaml
(1 + 2) * (3 + 4)
```

言語機能のデザインが済んだら、続いては「2. 抽象構文木の設計と実装」だ。

プログラム言語と呼ばれるものは総じて抽象構文木（AST; Abstract Syntax Tree）というツリー構造に変換される（インタープリタ系言語だとそうじゃないこともあるけど）。

例えば `1 + 2` という足算は以下のようになる。

![1+2の木構造](images/ast_simple.png)

抽象構文木のデータは次のようなイメージで定めておく。

```ts
// 2つの項の計算を表すノード
type BinaryExpressionNode = {
  kind: "BinaryExpression";
  operator: "add" | "multiply"; // 計算の種類. ここでは加算と乗算だけ
  left: ExpressionNode; // 演算子の左側の式
  right: ExpressionNode; // 演算子の右側の式
};

// 自然数を表すノード
type NatNode = {
  kind: "Nat";
  value: number; // そのノードが保持している値
};

// 式
type ExpressionNode = BinaryExpressionNode | NatNode;
```

「抽象」というのは、「ソースコードの全情報が保持されるわけではない」ということだ。 `1+2*3` も `1 + 2 * 3` も `1 + (2) * 3` も同じ形のデータ構造になる。一方で `1 + 2 * 3` と `(1 + 2) * 3` が同じ結果の構造になってしまったら計算を正しく行えない。この 2 つの式は、以下のような抽象構文木に変換されないといけない。

![優先度と木構造](images/ast_nested.png)

プログラミング言語をソースコードから抽象構文木に変換する機能のことを「構文解析器」とか「パーサー」と呼んだりする。たとえば先程の `(1 + 2) * 3` の変換処理であれば、 `parse` という関数を用意した上で以下のようなテストコートを書くことになる。

```ts
test("'(1 + 2) * 3' がパースできること", () => {
  expect(parse("(1 + 2) * 3")).toStrictEqual({
    kind: "BinaryExpression",
    operator: "multiply",
    left: {
      kind: "BinaryExpression",
      operator: "add",
      left: {
        kind: "Nat",
        value: 1
      },
      right: {
        kind: "Nat",
        value: 2
      }
    },
    right: {
      kind: "Nat",
      value: 3
    }
  });
});
```

ここではパーサーの実装コードそのものには触れないでおくが（長くなってしまうので）、実装のやり方や考え方だけ簡単に触れておく。

パーサーを用意するには、たとえば [PGE.js](https://pegjs.org/) のような「パーサージェネレータ」と呼ばれるライブラリを利用する。

ただ、この程度の自作言語であれば自分で書くのもそれほど難しくない。再帰下降構文解析と呼ばれるテクニックが常套手段として用いられる。

パーサージェネレータにせよ、再帰下降構文解析にせよ、「そのプログラミング言語で表現可能なコードを表現するための生成規則」を定義することになる。生成規則は BNF や EBNF と呼ばれる記述方法で表現されることが多い。いまここで扱っている「自然数の加算・乗算ができる言語」は以下のように表現できる。

```
expr  ::= mul ("+" mul)*
mul   ::= group ("*" group)*
group ::= "(" expr ")" | nat
nat   ::= "0" | "1" | "2" | ... | "127"
```

一番目の規則の `expr ::= mul ("+" mul)*` は加算を表していて「式は `mul` という規則の結果 1 つに続いて『`+` という記号と `mul` という規則の結果』が 0 回以上繰り返される」という意味を表している。`"+"` のように引用符付きで表しているのは、実際に言語中で利用される終端記号であり `("+" mul)*` の括弧とアスタリスクは「当該部分の繰り返し」の意味だ。同じように `mul` の規則は `group` という規則から生成されていて、こちらは乗算を表している。

`group` の規則は式を括弧でグルーピングするための規則だ。 `group ::= "(" expr ")" | nat` の `|` は、パイプ記号で区切ったどちらか一方の生成規則が適用されるという意味なので、`group` という生成規則は「 `expr` 規則を括弧で囲んだものか、もしくは、 `nat` という規則のどちらか」ということになる。 `nat` 規則はただの自然数を生成
する規則だ。

再帰下降構文解析は、BNF で表現された生成規則それぞれにパーサーを実装していくことで言語全体のパーサーを組み立てる手法だ。ここで例示した BNF から構文木を生成するパーサーの例を TypeScript Playground に書き残しておく。

[再帰下降構文解析の実装例](https://tsplay.dev/mLLXZm)

次に「3. 即時評価機能の実装」だ。

即時評価機能、という呼び方があっているのかわからないが、後続のコンパイラと区別するためにそう呼ぶことにする。「パースした構文木の式をその場で計算する」という意味で使っている。

構文木が完成していれば、木構造を再帰的に評価していくだけなので簡単。

```ts
function evaluate(expr: ExpressionNode): number {
  switch (expr.kind) {
    // 自然数ノードの場合、そのまま値を返す
    case "Nat": {
      return expr.value;
    }

    // 二項演算子ノードの場合、オペランドの評価結果から演算結果を返す
    case "BinaryExpression": {
      switch (expr.operator) {
        case "add": {
          return evaluate(expr.left) + evaluate(expr.right);
        }
        case "multiply": {
          return evaluate(expr.left) * evaluate(expr.right);
        }
      }
    }
  }
}
```

コンパイラを作る上では即時評価の機能は実装する必要はないのだけど「設計した言語を動かせるようにする」という楽しみが割と簡単に味わえるので、そういう意味ではオススメである。

最後は「4. コンパイル機能の実装」について。
