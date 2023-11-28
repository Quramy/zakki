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

_以下は deepl 翻訳結果_

> 8 月 3 日、WebAssembly CG は JavaScript の文字列セマンティクス/エンコーディングが Interface Types 提案の対象外であるかどうかについて投票を行う。この決定はおそらく Google、Mozilla、Bytecode Alliance/WASI によって支持されるでしょう。彼らは WebAssembly において C++、Rust、それぞれの非 Web セマンティクスやコンセプトを排他的に推進するという共通の関心を持っているようです。
>
> この投票が可決された場合、おそらく AssemblyScript は深刻な影響を受けるでしょう。この決定が JavaScript のような 16 ビット文字列セマンティクスを利用する言語（DOMString、C#、Java も参照）やそのユーザーに与える解決できない正しさやセキュリティの問題のために、AssemblyScript が開発したツールは最終的に非推奨とならざるを得ないからです。
>
> AssemblyScript がデータの整合性を保証するための唯一の実行可能な方法は、インターフェイス型への依存を参照型への依存に置き換え、標準ライブラリを JavaScript からインポートすることであると私たちは予想しています。すべての影響を知ることはできませんが、この移行は範囲が広すぎることが判明するかもしれませんし、もし可能であったとしても、Wasmtime や Wasmer のような WASI ホスト上で AssemblyScript を実行しているユーザーに影響を与える可能性があり、これらのプラットフォームでは JavaScript の標準ライブラリも GC も利用できないからです。その結果、これらのプラットフォーム上で将来のバージョンの AssemblyScript を利用することはおそらく不可能であり、以前のバージョンは安全ではないので避けることを強くお勧めします。
>
> 私たちは、結果として起こる Web プラットフォーム、プログラミング言語、セキュリティの崩壊、そして AssemblyScript プロジェクトにとっての特に不幸な結果を防ぐべきだと考えていますが、長年にわたる献身的かつ無駄な努力の結果、私たちは巨人に対して無力であると結論づけざるを得ません。

第一印象として、大手巨大企業の横暴により、規模は劣るものの頑張っている OSS コミュティや Web のエコシステムが侵犯されているようなことを書いているように見える。

どのようにしてこの結論（というか意思表明）に至ったのかが気になったので、自分の理解を深める上でこのメモを書くことにした。

### タイムライン的な話

- 7/26 AssemblyScript の WebSite に上述のバナーが掲載される
- 7/29 (理由は不明) このバナーの commit が revert されて見えなくなる
- 8/3 WebAssembly CG による投票が行われる。原案のまま可決される
- 8/3 AssemblyScript の WebSite から Core Team へのリンクが削除される

### 最初に断っておきたいこと

#### WASM の内部で利用される文字エンコードを決める話なの？

違う。「WASM が受け付けることのできる（または WASM から出力することのできる）文字って何なんだ？」を決めたい、という話。

対象は WebAssembly Module の境界部分であって、WebAssembly Module 内部の話はまったく関係ない。

そもそも現状の WASM Spec でも文字列が登場する部分は `import` や `export` 、Custom Name Section のために存在しているし、それらで利用可能な文字の形式は UTF-8 であると定められている。

#### AssemblyScript 死んじゃったの？

知らない。でも多分死んだわけではないと思う。 AssemblyScript が出力した WASM のモジュールだってちゃんと動作するし、そこには快適変更を加えるような話ではない。

> And we're dead. Congrats.

も含めてツイ消しされちゃったし。

## 背景の背景 1: 文字列についてのおさらい

### Unicode

Unicode は文字 1 つ 1 つに、対応する整数値をマッピングしている。マッピングされた整数値のことを符号位置（コードポイントとも）と呼ぶ。

- `U+0000` ~ `U+D7FF`, `U+E000` ~ `U+FFFF`: 基本多言語面(BMP; Basic Multilingual Plane).
- `U+10000` ~ `U+10FFFF`: 追加面
  - e.g. `U+1F700` ~ `UL1F773` は錬金術記号であり、追加多言語方面(SMP; Supplementary Multilingual Plane)として Unicode 第 1 面に記載されている.

BMP で歯抜けている範囲( `U+D800` ~ `U+DFFF` )を代理符号位置と呼ぶ。この範囲にマッピングされる文字は存在しない。

### UTF-16

Unicode におけるエンコード方式の 1 つが UTF-16

16 という数字にもあるとおり、基本的に 16 bit (= 2 byte) で 1 つの文字を表すことができる。

```js
"\u{0061}"; // a
"\u0061"; // a

"\u{3042}"; // あ
"\u3042"; // あ
```

「基本的に」と言っているのは、「Unicode におけるよく利用する文字に限って言えば」という但し書きがつくため。

BMP の範囲外、即ち追加面の文字はコードポイントの整数値が 4 byte を超えてしまうため、2 byte + 2 byte = 4 byte を用いて表すことになる。

```js
"\u{1F4A9}"; // 💩
"\uD83D\uDCA9"; // 💩
```

これが「サロゲートペア」と呼ばれる文字。代理符号位置に収まるコードユニットを 2 つ組み合わせることで 1 文字を表現する。

### プログラミング言語の内部表現と UTF-16

JavaScript や DOMString は文字列が UTF-16 のコードユニット列として管理されている。

> BMP で歯抜けている範囲( `U+D800` ~ `U+DFFF` )を代理符号位置と呼ぶ。この範囲にマッピングされる文字は存在しない。

と書いたが、JavaScript にせよ、DOMString にせよ、この範囲上のコードユニットを単独で文字列として扱えてしまう。

```ts
console.log("\uDEAD"); // ????
```

「UTF-16 で利用可能なコードユニットの列」が必ずしも UTF-16 として正しいわけではないのがポイント。

> Depending on the programming environment, a Unicode string may or may not be required to be in the corresponding Unicode encoding form. For example, strings in Java, C#, or ECMAScript are Unicode 16-bit strings, but are not necessarily well-formed UTF16 sequences. In normal processing, it can be far more efficient to allow such strings to contain code unit sequences that are not well-formed UTF-16—that is, isolated surrogates.
> Because strings are such a fundamental component of every program, checking for isolated surrogates in every operation that modifies strings can create significant overhead, especially because supplementary characters are extremely rare as a percentage of overall text in programs worldwide.

> プログラミング環境によっては、Unicode 文字列が対応する Unicode エンコーディング形式である必要がある場合とない場合があります。たとえば、Java、C#、ECMAScript の文字列は、Unicode の 16 ビット文字列ですが、必ずしも整形された UTF16 シーケンスではありません。通常の処理では、このような文字列に、整形されていない UTF-16 のコードユニットシーケンス、つまり孤立したサロゲートを含めることを許可したほうが、はるかに効率的な場合があります。
> 文字列はすべてのプログラムの基本的な構成要素であるため、文字列を変更するすべての操作で孤立したサロゲートをチェックすると、かなりのオーバーヘッドが発生します。

https://www.unicode.org/versions/Unicode13.0.0 2.7 節より

ちなみに、このような独立したサロゲートコードユニットを許容してしまうような文字列のエンコード形式をさして WTF-16 と呼ぶことがある。

WTF-16 の場合、他の Unicode のエンコード方式の世界に持ちこむ（持ってくる）ような処理をした場合に、問題が生じる場合がある。

例えば以下のコードを実行すると `false` という結果が帰ってくるはず。

```js
// WTF 文字列 -> UTF-8 (ArrayBuffer) -> UTF-16 文字列の例
new TextDecoder().decode(new TextEncoder().encode("\uDEAD")) === "\uDEAD";
```

`TextEncoder` はエンコードする際に孤立したサロゲートコードユニットを `U+FFFD` で置換するため、さらに `TextDecoder` で decode しても同じ文字にはならない（逆に以下は真となる）。

```js
new TextDecoder().decode(new TextEncoder().encode("\uDEAD")) === "\uFFFD"; // true
```

この挙動は [WHATWG Encoding](https://encoding.spec.whatwg.org/#encoders-and-decoders) で定義されているが、同じ UTF-8 対象であっても [JSON](https://datatracker.ietf.org/doc/html/rfc8259#section-8.2) は仕様としては "unpredictable" 、すなわち「何が起きるか保証できません」というスタンス。

この話の文脈としては「内部表現に WTF-16 が利用されている場合、他の符号化を採用しているプラットフォームとの相互運用性に問題がある」という点さえ抑えておけばよいはず。

## WebAssembly

ここからが WASM の話。

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

### 背景の背景 2: Module Linking

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

  ;; この部分が Core Module
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

- 従来の WASM MVP Spec で記述される部分: Core Module
- `adapter_module` によってくるまれた部分: Adapter Module

Adapter Module によって Link されたひと塊のことを "Component Model" と呼ぶ（スライドの p23 あたり）。

### Interface Types のモチベーション

MVP における WebAssembly においては、値として表現できる型（ここでは関数の引数、戻り値として利用可能な型）は以下だけ。

- Number Type: `i32`, `i64`, `f32`, `f64`
- Reference Type: `funcref` , `externref` .

Char も Tuple も Array も List もない。

List 様なデータ構造をどう表現しているかというと、線形メモリ上に自分でポインタを駆使したデータ構造を作っていく必要がある（なんなら `malloc` から自作することになる）。

- 現状の WASM の表現力の貧弱さが、WASI のような Interface の抽象化の妨げになっていくため、よりハイレベルな値のやりとりをできるようにしたい
- WASM からの JS API 呼び出しが高コスト。実際は数値を渡すだけのはずが、Web IDL として定義された API を通して ECMAScript Binding を実行する羽目になっている
  - 特定の Object を特定の方法で import したときに高速動作するような v8 オプションがあったりするらしいが、それと関連した話だろうか...? e.g. `--wasm-math-intrinsics`

### WebAssembly CG のやろうとしていること

- Server Side WASM (e.g. Lucet など) などで、WASM は Web 以外でも動作させたい欲求が出てきた. JS API さえ呼べればいいというわけではなくなった.
  - サーバー上で WASM を動作させる上で、ホストとのやりとりを標準化する必要がある。これが WebAssembly System Interface(WASI) 。POSIX の WASM 版的なイメージ。
- WASI を定める上で、Interface となる Type が WASM に足りていない
  - POSIX の `open()` 相当を定義しようにも、 `char* pathname` どうするんだ、となる
  - 現行の MVP でやるのであれば、線形メモリと `i32` offset 値を介する程度しか解がない
  - interface のためだけに、呼び出し元モジュールの線形メモリを export したくない
  - ホスト側の文字列エンコードを呼び出し側が意識する必要がある
- 線形メモリに寄らないより高位な型の表現が WebAssembly Spec に必要となる. これを規定するのが Interface Type Spec
- Interface Types は Value Types に求められる性質が異なる
  - Serialize 可能でなければならない. Reference Types も含めた Value Types はこれを満たしていない
  - interface している最中に o(n) のリソース消費があってはならない. アフィン性(At most once)を強制したい
  - e.g. `local.get` などは使ってはいけない
- したがって、Interface Types に求められるのは Value Types とは異なる型のセット、MVP WASM よりも狭い Instructions のセット
  - このような変換に特化した関数を定義するために用意されるのが `(adapter_func ...)`.
- Adapter Function はどこに定義できるのか？を含め、WASM にレイヤー構造を導入しようとしているのが Module Linking Spec
  - Linking Layer, Adapter Layer, Core Layer の 3 層構造. ネットワークのレイヤーモデルみたいなもの
  - Core Layer は MVP レベルの WASM モジュールをそのまま使える(変更がない)ようにしている。import や export でホストの追加情報を知らなくてもよい構成
  - これは WASI が目指している Host Agnostic の文脈とも相性がよい
  - Adapter Function、すなわち Interface Types が登場できるのは
  - 従来の Post MVP としても、ES Module のように WASM Module 同士を直接 import/export して使いたい、という要求は上がっていた
  - ここには当然、異なる言語から Compile された WASM Module 同士のリンクも含んでいる
- つまるところ、Module Linking x Interface Types の組み合わせで、Host Agnostic x Language Agnostic で高速かつ可搬性のある WASM Component Model が実現できるはず
  - Core Module に変更を加えず、外界の自由度を上げよう、という話をしている
  - この "WASM Component Model" の MVP 達成で一番ボトルネックと予見されるのが、中間層である Adapter Function の仕様化
- Component Model の MVP Adapter Functions の議論を待たずに、Module Linking と Interface Types だけをまず決めてしまいたい
- Host 側で WASM のバイナリ構造を意識しつつ、ハイレベルなデータ型を WASM の Value Type 相当に変換する仕様を決めにいく流れになった
  - これが "Canonical ABI" と呼ばれている部分
  - ABI は WASM 「Interface Types として値を渡すときにどのようにしたらよいか」という標準仕様とみなすことができる
  - 実装は各 tool chain に任せるスタンス。例えば Rust の wasm-bindgen が js 側の生成物が ABI に相当するようになるはず
- ABI を定める上で、Interface Types に含まれる文字列の定義をしていく流れに
  - エンコード(UTF-16 or UTF-8)は論点ではない
  - セマンティクス、すなわち「Interface を通して WASM に渡せる文字とは？」が問題

### CG 側の意見

https://github.com/WebAssembly/interface-types/issues/135

- WTF-16 の範囲をセマンティクスとすると、C/C++ や Go, Rust などの言語で WASM から振ってくる孤立したサロゲートに悩まされることになる
- [WHATWG Encoding](https://encoding.spec.whatwg.org/#encoders-and-decoders) とか、既に DOMString / JavaScript から孤立したサロゲートを持ち出させない機能は色々あるし、今回の ABI もそれと同じはず
- 仮に WASM に WTF-16 を渡せるようにしても、WASM 側で trap される可能性が無いわけではないから、(JavaScript のような) 言語にとっても幸福なこととは言えないはずだ

### AssemblyScript 側の意見

- Round Trip で情報の損失が起こり、JavaScriptやWebの世界に重大な問題が発生する

```ts
// WTF 文字列 -> UTF-8 (ArrayBuffer) -> UTF-16 文字列の例
new TextDecoder().decode(new TextEncoder().encode("\uDEAD")) === "\uDEAD";
```

↑ と同じだけでは？という気がするのだけど、何が問題なのかがわからず。

## 所感

> This decision will likely be backed by Google, Mozilla and the Bytecode Alliance/WASI, who appear to have a common interest to exclusively promote C++, Rust respectively non-Web semantics and concepts in WebAssembly
>
> (中略)
>
> We believe that the resulting Web platform, programming language and security breakage, as well as the particularly unfortunate outcome for the AssemblyScript project, should be prevented, but after many years of dedicated yet futile efforts we also have to conclude that we are powerless against giants.

諸々調べた上で感じるのは、ASのWebsiteに掲載されていたこの言い回しには大分語弊があるのでは、と思う。

確かに Interface Types のモチベーションの一つにWASI文脈あるのは事実だが、「Tech Giantsに力が及ばなかった」と捉えるものでもないのでは。

https://github.com/AssemblyScript/website/commit/5c7a9d2f3df5a5083315e4880b747f8f8c3c1004

## References

- https://github.com/WebAssembly/meetings/blob/main/main/2021/CG-08-03.md : WASM CG の Meeting Agenda
- https://www.unicode.org/versions/Unicode13.0.0
- https://unicode.org/charts/PDF/U1F700.pdf : 錬金術記号の定義
- https://heycam.github.io/webidl/#idl-DOMString : DOM String の定義
- https://www.w3.org/TR/DOM-Level-3-Core//core#ID-C74D1578 : DOM String の定義由来である DOM Lv 3 の定義
- https://datatracker.ietf.org/doc/html/rfc8259#section-8.2 : JSON の RFC で isolated surrogate が来た場合の動作についての言及箇所
- https://encoding.spec.whatwg.org/#encoders-and-decoders
- https://hacks.mozilla.org/2018/10/webassemblys-post-mvp-future/ : WASM の POST MVP に対する概略. 2018 年の記事なので結構古い.
- https://docs.google.com/presentation/d/1PSC3Q5oFsJEaYyV5lNJvVgh-SNxhySWUqZ6puyojMi8/edit#slide=id.gceaf867ebf_0_147 : Layering Module Linking Proposal の slide
- https://webassembly.github.io/spec/core/syntax/types.html#value-types : 現行の WASM における Value Types
- https://docs.google.com/presentation/d/1qVbBsDFmremBGVKiOAzRk7svjinNq6LXfJ1DzeFwKtc/edit#slide=id.p :
- https://webassembly.github.io/spec/core/syntax/values.html#names : WASM の name に関する定義
