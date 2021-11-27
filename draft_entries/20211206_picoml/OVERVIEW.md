# 自作プログラミング言語とコンパイラの話

## PicoML とはなにか

- 自作プログラミング言語. ML ベース. OCaml サブセット
- Node.js 上での実行
- WebAssembly をターゲットにしたコンパイラ
- Web Playground
- CLI
  - REPL
  - Compiler
- TypeScript Only, Zero dependencies
- 本体 10,000 loc, 全体で 15,000 loc
- PicoML で動作するコードの例

## PicoML を作った理由

- 型推論を学んでみたかった
- フルスクラッチで自作言語実装をしてみたかった
  - TS とか AST とか普段から言っていることに対する負い目
- 大学でできなかったことのおさらい

## 工期

- 2021.03.08 - 2021.04.03: 評価, 型推論
- 2021.04.03 - 2021.05.07: WebAssembly コンパイラ
- 2021.05.12 - 2021.05.21: Web Playground UI
- 2021.07.16 - 2021.07.22: 言語機能の拡張

## 得たもの

- プログラムを書くことそのものの楽しさ
- 知識の再確認
  - TypeScript のテクニック
- 新しい言語の知識
- WebAssembly 知識
  - WAT が読める・かけるようになった
  - Module や Adapter Spec Proposal 読んだり

## 設計と実装

## プログラミング言語をつくる流れ

- Design
- Abstract Syntax Tree
- Evaluate expression immediately
- Separate Frontend and Backend
- Compile to WASM Stack machine
- Error reporting via type system

## パッケージの全体像

(Module Dependencies Graphics)

## 個別モジュールの概要

- Compiler Frontend
  - 再起下降パーサー
  - パーサーコンビネータ
  - Let 多相と単一化
- Compiler Backend
  - WebAssembly Infrastructure
    - WebAssembly Text Format / WebAssembly Structure / WebAssembly Binay Format
      - WAT as Intermediate representation
      - Abbreviation, S Expression
    - WAT Code Template
    - Footprint Module
  - Compiler Technique
    - 環境と De Bruijn Indexer
    - ML Function と WebAssembly Table
    - ML Value representation in WebAssembly
    - Polymorphic Operation Dispatch Dynamically / Statically

## 今後取り組みたい機能

- GC
- 最適化

## 参考にしたもの

- [プログラミング言語の基礎概念](https://www.amazon.co.jp/%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0%E8%A8%80%E8%AA%9E%E3%81%AE%E5%9F%BA%E7%A4%8E%E6%A6%82%E5%BF%B5-%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E6%83%85%E5%A0%B1%E5%AD%A6%E3%82%B3%E3%82%A2%E3%83%BB%E3%83%86%E3%82%AD%E3%82%B9%E3%83%88-%E4%BA%94%E5%8D%81%E5%B5%90-%E6%B7%B3/dp/4781912850)
- [低レイヤを知りたい人のための C コンパイラ作成入門](https://www.sigbus.info/compilerbook)
- [The OCaml Language](https://ocaml.org/manual/language.html)
- [Real World OCaml](https://dev.realworldocaml.org/)
- [WebAssembly テキストフォーマットを理解する | MDN](https://developer.mozilla.org/ja/docs/WebAssembly/Understanding_the_text_format)
- [WebAssembly Specifications](https://webassembly.github.io/spec/core/)
