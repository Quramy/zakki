# WebAssembly における Interface Types と文字列表現の騒動についての理解とまとめ

## はじめに

### なぜこれを書こうと思ったか

7/27 頃に AssemblyScript のサイトに Up されたのを見て興味をもつ。

> On August 3rd, the WebAssembly CG will poll on whether JavaScript string semantics/encoding are out of scope of the Interface Types proposal. This decision will likely be backed by Google, Mozilla and the Bytecode Alliance/WASI, who appear to have a common interest to exclusively promote C++, Rust respectively non-Web semantics and concepts in WebAssembly.
>
> If the poll passes, which is likely, AssemblyScript will be severely impacted as the tools it has developed must eventually be deprecated due to unresolvable correctness and security problems the decision imposes upon languages utilizing JavaScript-like 16-bit string semantics (see DOMString, also C#, Java) and its users.

> It is our expectation that AssemblyScript's only viable way forward to guarantee data integrity will be to replace its dependency upon Interface Types with a dependency upon Reference Types and import its standard library from JavaScript. While the full impact cannot be known, this transition may either turn out to be too large in scope, or, if it can be done, is likely to impact users running AssemblyScript on WASI hosts like Wasmtime and Wasmer, in that neither the JavaScript standard library nor a GC will be available on these platforms. As a result, it would likely not be feasible anymore to utilize future versions of AssemblyScript on these platforms, and we would strongly recommend to avoid earlier versions since these will not be safe.
>
> We believe that the resulting Web platform, programming language and security breakage, as well as the particularly unfortunate outcome for the AssemblyScript project, should be prevented, but after many years of dedicated yet futile efforts we also have to conclude that we are powerless against giants.

https://github.com/WebAssembly/interface-types/issues/135

https://twitter.com/dcodeIO/status/1422610330803396614

> And we're dead. Congrats.

## 文字列についてのおさらい

### UTF-16

2byte(16 bit) または 4(32 bit)で 1 文字を表す文字列の符号化方式。

- `U+0000` ~ `U+D7FF`, `U+E000` ~ `U+FFFF`: 基本多言語面(BMP; Basic Multilingual Plane).
- `U+10000` ~ `U+10FFFF`: 追加面. **いわゆるサロゲートペアで表現されるのはここ**.
  - e.g. `U+1F700` ~ `UL1F773` は錬金術記号であり、追加多言語方面(SMP; Supplementary Multilingual Plane)として Unicode 第 1 面に記載されている.

BMP に含まれていない符号( `U+D800` ~ `U+DFFF` )を代用符号位置と呼ぶ。well-formed な UTF-16 では、代用符号が単独で存在することを許容しない。

### プログラミング言語における文字列の内部表現

文字列のメモリ表現に Unicode のコードポイント列を採用しているプラットフォームは色々ある。Web エンジニアが日頃触れているプラットフォームだと以下。

- [ECMAScript]()
- [DOMString](https://heycam.github.io/webidl/#idl-DOMString)

```js
"\u0061"; // a
"\u3042"; // あ
"\uD83D\uDCA9"; // 💩
```

代用符号はサロゲートペアの形式でしか意味ないのであれば、次のようなコードはそもそも文字列を生成しないのか？

答えは No であり、例えば以下の JavaScript は普通に実行できる。すなわち「孤立した Surrogate Code Unit が内部的に存在しうる」言語である。

```ts
console.log("\uDEAD"); // ????
```

「Unicode のコードポイント列」はそれ即ち UTF-16 にならないのがポイント。

> Depending on the programming environment, a Unicode string may or may not be required to be in the corresponding Unicode encoding form. For example, strings in Java, C#, or ECMAScript are Unicode 16-bit strings, but are not necessarily well-formed UTF16 sequences. In normal processing, it can be far more efficient to allow such strings to contain code unit sequences that are not well-formed UTF-16—that is, isolated surrogates.
> Because strings are such a fundamental component of every program, checking for isolated surrogates in every operation that modifies strings can create significant overhead, especially because supplementary characters are extremely rare as a percentage of overall text in programs worldwide.

> プログラミング環境によっては、Unicode 文字列が対応する Unicode エンコーディング形式である必要がある場合とない場合があります。たとえば、Java、C#、ECMAScript の文字列は、Unicode の 16 ビット文字列ですが、必ずしも整形された UTF16 シーケンスではありません。通常の処理では、このような文字列に、整形されていない UTF-16 のコードユニットシーケンス、つまり孤立したサロゲートを含めることを許可したほうが、はるかに効率的な場合があります。
> 文字列はすべてのプログラムの基本的な構成要素であるため、文字列を変更するすべての操作で孤立したサロゲートをチェックすると、かなりのオーバーヘッドが発生します。

https://www.unicode.org/versions/Unicode13.0.0 2.7 節より

ちなみに Rust だと以下は Compile すら通らない（そもそも `String` も `str` も UTF-8）。

```rust
println!("\u{DEAD}");
```

### UTF-8 との相互運用性

UTF-8 は最大で 4byte まで利用する符号化方式のため、UTF-16 のサロゲートペアの概念はそもそも存在しない。

"well-formed な" UTF-16 であれば、UTF-8 とは相互に行き来が可能である。

問題は "well-formed でない" すなわち WTF との相互運用のケース。

WTF の例として JavaScript で考える。

```js
// WTF 文字列 -> UTF-8 (ArrayBuffer) -> UTF-16 文字列の例
new TextDecoder().decode(new TextEncoder().encode("\uDEAD")) === "\uDEAD";
```

上記のコードを実行すると `false` となるはず。

`TextEncoder` はエンコードする際に孤立したサロゲートコードユニットを `U+FFFD` で置換するため、さらに `TextDecoder` で decode しても同じ文字にはならない（逆に以下は真となる）。

```js
new TextDecoder().decode(new TextEncoder().encode("\uDEAD")) === "\uFFFD"; // true
```

この挙動は [WHATWG Encoding](https://encoding.spec.whatwg.org/#encoders-and-decoders) で定義されているが、同じ UTF-8 対象であっても [JSON](https://datatracker.ietf.org/doc/html/rfc8259#section-8.2) は仕様としては "unpredictable" 、すなわち「何が起きるか保証できません」というスタンス。

この話の文脈としては「内部表現に WTF が利用されている場合、他の符号化を採用しているプラットフォームとの相互運用性に問題がある」という点を抑えておけばよさそう。

## WebAssembly

### WASM の基本

```wat
(module
  ;; 引数を2倍する関数の例
  (func $main (param $a i32) (result i32)
    local.get $a
    i32.const 2
    i32.mul
  )
  (export "main" (func $main))
)
```

```js
const source = await fs.readFile("module.wasm");
const { instance } = await WebAssembly.instantiate(source);
console.log(instance.exports["main"](10)); // 20
```

### WASM はまだまだ MVP

### Module Linking

今回の UTF の件の背景となるのは Module Linking の proposal 見直しの件。

[Scoping and Layering the Module Linking and Interface Types proposals](https://docs.google.com/presentation/d/1PSC3Q5oFsJEaYyV5lNJvVgh-SNxhySWUqZ6puyojMi8/edit#slide=id.p)

```wat
(module $APP
  (import "libc" (module $LIBC
    (export "malloc" (func (param i32) (result i32)))
  ))
  (instance $libc (instantiate $LIBC))
  (module $CODE
    (import "libc" (instance $libc
      (export "malloc" (func (param i32) (result i32)))
    ))
    (func (export "run") (param i32 i32) …)
  )
  (instance $code (instantiate $CODE (import "libc" (instance $libc))))
  (func (export "run") (param i32 i32)
    (call (func $code "run") …)
  )
)
```

(スライドの p3 より)

すごく雑にいうと、「 `WebAssembly.instantiate` の結果得られるであろう Type 」という表現を持ち込むことで、WASM 上から他のモジュールの機能を呼び出そうね、という考え方。

WASM のホストに依存している（多分 "libc" のような Module Specifier 的な部分のことを言っている？）

これに対して "Adapter Module" という名前で、instantiate 化などの責務を明示的に分離する案が、スライドの p6 にかかれている内容。
ポイントは `$CODE` の Module 以下は従来の MVP レベルの spec で表現できており、(JS で instantiate するときと同じ様に) "libc.malloc" が使えるように見える点。

```
(adapter_module
  (import "libc" (module $LIBC …))
  (instance $libc (instantiate $LIBC))

  (module $CODE
    (import "libc" "malloc" (func …))
    …
    (func (export "run") …)
  )

  (instance $code (instantiate $CODE
    (import "libc" "malloc" (func $libc "malloc"))
  ))
  (export "run" (func $code "run"))
)
```

### Interface Types のモチベーション

MVP における WebAssembly においては、値として表現できる型（ここでは関数の引数、戻り値として利用可能な型）は以下だけ。

- Number Type: `i32`, `i64`, `f32`, `f64`
- Reference Type: `funcref` , `externref` .

Char も Tuple も Array も List もない。

List 様なデータ構造をどう表現しているかというと、線形メモリ上に自分でポインタを駆使したデータ構造を作っていく必要がある（なんなら `malloc` から自作することになる）。

- 現状の WASM の表現力の貧弱さが、WASI のような Interface の抽象化の妨げになっていくため、よりハイレベルな値のやりとりをできるようにしたい
- WASM からの JS API 呼び出しが高コスト。実際は数値を渡すだけのはずが、Web IDL として定義された API を通して ECMAScript Binding を実行する羽目になっている
  - 特定の Object を特定の方法で import したときに高速動作するような v8 オプションがあったりする e.g. `--wasm-math-intrinsics`

### Interface Types とその他の Proposal の関係

## References

- https://www.unicode.org/versions/Unicode13.0.0
- https://unicode.org/charts/PDF/U1F700.pdf : 錬金術記号の定義
- https://heycam.github.io/webidl/#idl-DOMString : DOM String の定義
- https://www.w3.org/TR/DOM-Level-3-Core//core#ID-C74D1578 : DOM String の定義由来である DOM Lv 3 の定義
- https://datatracker.ietf.org/doc/html/rfc8259#section-8.2 : JSON の RFC で isolated surrogate が来た場合の動作についての言及箇所
- https://encoding.spec.whatwg.org/#encoders-and-decoders
- https://hacks.mozilla.org/2018/10/webassemblys-post-mvp-future/ : WASM の POST MVP に対する概略. 2018 年の記事なので結構古い.
- https://docs.google.com/presentation/d/1PSC3Q5oFsJEaYyV5lNJvVgh-SNxhySWUqZ6puyojMi8/edit#slide=id.gceaf867ebf_0_147 : Layering Module Linking Proposal の slide
- https://webassembly.github.io/spec/core/syntax/types.html#value-types : 現行の WASM における Value Types
