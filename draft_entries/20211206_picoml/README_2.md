## Intermediate Representation

ここからはもう少し具体的なテクニックの話を書いていく。

「プログラミング言語をつくる流れ」の節では、以下の流れで実装を説明していた。

```
Source code
  |
  | (parse)
  v
Abstact Syntax Tree
  |
  | (compile)
  v
WebAssembly Binary Format(wasm)
```

PicoML の実装にあたっては、このフローはもう少し細分化している。

```
Source code
  |
  | (parse)
  v
ML Abstact Syntax Tree
  |
  | (compile)
  v
WAT Abstact Syntax Tree
  |
  | (convert)
  v
WebAssembly Strutual Object
  |
  | (unparse)
  v
WebAssembly Binary Format(wasm)
```

ML の AST は WebAssembly Text Format(WAT) の AST に変換に徹しているところがポイント。

WASM は [WebAssembly Structure](https://webassembly.github.io/spec/core/syntax/index.html) をバイナリ表現に落とし込んだものなのだけど、コンパイルロジックの中間構造として扱うには悩ましい点が多いのだ。

具体例で説明した方が伝わりやすいかもしれない。次のコードは WAT としては正しいコード例だ。WebAssembly に明るくなくても、なんとなく読めるのではないだろうか。

```wat
(module
  (func $twice (param $x i32) (result i32)
    local.get $x
    i32.const 2
    i32.mul)
  (func (export "main") (result i32)
    i32.const 2
    call $twice))
```

`$twice` のような名前の部分や、関数シグネチャを関数名の直後に記述できているのは飽くまで Abbreviation (略記法)によるもので、WebAssembly Structure の世界には、それらの概念は存在しない。関数やローカル変数を指定するのに利用可能なものは順番を示すインデックス値だけだし、関数シグネチャも `func` に直接記述できず `type` という構造に分けて表現しなくてはならない。

Abbreviation を一切使わずに先程の WAT を書き直すと、すなわちこれが Structure に近い姿ということになるのだけど、次のようになってしまう。

```wat
(module
  (type (func (result i32)))
  (type (func (param i32) (result i32)))
  (func (type 1)
    local.get 0
    i32.const 2
    i32.mul)
  (func (type 0)
    i32.const 2
    call 0)
  (export "main" (func 1)))
```

WebAssembly Structure をコンパイルロジックの中間表現に用いることもできなくはないが、`call $twice` に相当する ML ノードを処理する際に Structure に `$main` という名前は保持されないので、結局 ML AST の探索中は常に「`$twice` は `func 0` である」というインデックス管理する羽目になる。

別途自前でインデックスの管理をするくらいであれば、ML AST のコンパイル時は Abbreviation を許容した WAT AST をそのまま中間表現としておき、一通り ML -> WAT AST のコンパイルが済んだ後で Abbreviation を正規化して Structure に落とした方が断然見通しが良いのだ。

また、このように ML AST -> WAT AST -> WebAssembly Structure -> WASM として、変換を多段階にしておくことによって、「整数定数をセットするには `0x41`」のようなことを、ML 部分のコンパイル時に考えなくて済むようになり、コード全体の可読性が向上するという利点もあった。

## WAT Parser / Unparser / Template

最終的に WASM を出力する必要があるので、 WAT AST -> WASM の変換処理を実装が生じるのは上述の通りだ。

これとは別に、PicoML には以下の実装も含まれている。

- WAT AST から WAT ソースコード文字列を出力するための unparser
- WAT のソースコード文字列から WAT AST を得るための parser

両方とも ML ソースコードから WASM を得る目的のみであれば不要だったのだけれど、副次的な理由があって実装することになった。

まず WAT AST to WAT Source の unparser だが、これは主に自分用のデバッグ目的。

コンパイル処理を書いていく中で、コンパイル結果が誤っている際にどこに間違いがあるのかを追おうとすると、やはり Intermediate Representation である WAT AST を文字列で読めないと困るからだ。

折角 unparser を用意したので、賑やかし的に Web Playground でも WAT のソースコードも表示するようにしている。

次に WAT Source -> WAT AST の parser について。

ML コードから WebAssembly の生成において、純粋にすべての WAT AST が ML AST に対応づくわけではない。対象の処理によっては、ヘルパー的な WebAssembly 関数に頼らなくてはならない。

たとえば、リストであったり変数環境といったデータ構造は、スタック上に保持するわけにはいかないので、WebAssembly の線形メモリに格納することになる。そうなると、C 言語でいうところの `malloc` 関数、すなわちメモリアロケータが必要になってくる。以下は実際に PicoML で使っているメモリアロケータ用のヘルパー関数（ちなみに、PicoML は現状では GC を持っておらず、線形メモリを食いつぶしていくだけの設計になっている）。

```wat
(module
  (memory $__alloc_mem__ 10)
  (global $__alloc_st__ (mut i32)
    i32.const 4)
  (func $__malloc__ (param $size i32) (result i32) (local $next i32)
    global.get $__alloc_st__
    local.set $next
    global.get $__alloc_st__
    local.get $size
    i32.add
    global.set $__alloc_st__
    local.get $next))
```

このようなヘルパー関数も、PicoML の中では WAT 文字列で管理しておき、それを一度 WAT AST にパースしたのち、必要に応じて最終的な出力にバンドルするようにしている。原理的には WAT AST の木構造だけ JSON か何かで持っておけば十分なのだけど、ここもやはり読みやすい形式で管理しておいた方が楽なのだ。

また、PicoML のプログラムコードを AST に変換する機構を作る過程で、パーサーコンビネータは実装しているので、それらを再利用すれば WAT AST parser もさして大変な作業ではない。

WAT Parser がパーサーコンビネータから作られているというのは、ヘルパー関数以外の場所でも役に立っている。

AST Template を用意しておくことで、 コンパイルのメイン部分、すなわち ML AST -> WAT AST の変換処理も WAT での表現に近い状態でソースが書けるのだ。

「プログラミング言語をつくる流れ」の中で記述した、以下のコンパイル処理を再び例に取り上げてみる。

```ts
function compileToInstructions(
  expr: ExpressionNode,
  instructions: number[] = []
): number[] {
  switch (expr.kind) {
    case "Nat": {
      return [...instructions, 0x41, expr.value];
    }
    case "BinaryExpression": {
      switch (expr.operator) {
        case "add": {
          return [
            ...instructions,
            ...compileToInstructions(expr.left, instructions),
            ...compileToInstructions(expr.right, instructions),
            0x6a
          ];
        }
        case "multiply": {
          return [
            ...instructions,
            ...compileToInstructions(expr.left, instructions),
            ...compileToInstructions(expr.right, instructions),
            0x6c
          ];
        }
      }
    }
  }
}
```

これを AST テンプレートで記述しなおすと、次のようになる。`wat.instructions` という Tag Function が「WebAssembly Instruction の AST を生成する関数」を作るようにしている。

```ts
function compileToInstructions(
  expr: ExpressionNode,
  instructions: () => WATInstructionNode[]
): () => WATInstructionNode[] {
  switch (expr.kind) {
    case "Nat": {
      return wat.instructions`
        i32.const ${expr.value}
      `;
    }
    case "BinaryExpression": {
      switch (expr.operator) {
        case "add": {
          return wat.instructions`
            ${instructions}
            ${compileToInstructions(expr.left, instructions)}
            ${compileToInstructions(expr.right, instructions)}
            i32.add
          `;
        }
        case "multiply": {
          return wat.instructions`
            ${instructions}
            ${compileToInstructions(expr.left, instructions)}
            ${compileToInstructions(expr.right, instructions)}
            i32.mul
          `;
        }
      }
    }
  }
}
```

二項演算だとそもそもの変換コードがシンプルなので、あまり旨味が伝わらないかもしれない。しかし、関数適用や let 式などは生成する WAT がもう少し複雑になってくるため、そういった変換処理であっても、コンパイラのソースコード上では WAT がイメージしやすいようにできている。

「コンパイル中にコンパイル後の IR をテンプレートで管理できるといいな」というのは、何も PicoML で思いついた話ではなく [@babel/template](https://babel.dev/docs/en/babel-template) を Babel Plugin で利用したときの記憶が残っていたので、その体験を生かしたに過ぎない。
