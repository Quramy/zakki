# TypeScript 4.2 RC Released

TypeScript 4.2 RC がリリースされています。

https://devblogs.microsoft.com/typescript/announcing-typescript-4-2-rc/

## New Features

### Leading/Middle Rest Elements in Tuple Types

下記のように、Rest Elements が tuple の最初でなくても使えるようになっています。

```ts
let foo: [...string[], number];

foo = [123];
foo = ["hello", 123];
foo = ["hello!", "hello!", "hello!", 123];

let bar: [boolean, ...string[], boolean];

bar = [true, false];
bar = [true, "some text", false];
bar = [true, "some", "separated", "text", false];
```

以下のような、「最後の引数だけ意味が違う」ような関数定義が書ける、ということですね。

```ts
declare function doStuff(
  ...args: [...names: string[], shouldCapitalize: boolean]
): void;
```

最初から TypeScript で関数を書いてるときは、あまり上記のような仮引数定義をしないと思いますが、既存の js 関数がこういう形式をしていた場合、d.ts を書くテクニックとして頭の片隅に置いておくとよいでしょう。

### Smarter Type Alias Preservation

Type Alias を合成して得られる合成型について、Error メッセージや VSC での hover したときの tooltip が少し見やすくなった、という話です。

以前より、以下のような書き方をした場合に、 変数 `x` での tooltip は `BasicPrimitive` と表示されるようになっていました（いつからだったか覚えてないけど、結構前から）。

```ts
export type BasicPrimitive = number | string | boolean;

export function doStuff(value: BasicPrimitive) {
  let x = value;
  return x;
}
```

ただ、下記のように、 `BasicPrimitive` だけでなく、他の型が混ざったような推論結果（以下でいうところの `duStuff` の戻り値型など）については、TypeScript 4.1 以前では `number | string | boolean | undefined` のように「バラされた状態」として表示されてしまいます。

```ts
export type BasicPrimitive = number | string | boolean;

export function doStuff(value: BasicPrimitive) {
  if (Math.random() < 0.5) {
    return undefined;
  }
  return value;
}
```

TypeScript 4.2 ではこれを改善し、 `BasicPrimitive | undefined` と表示できるようになった、ということです。

tooltip はともかくとして、tsc のエラーメッセージが読みやすくなる可能性があるのは嬉しいですね。
ただ、僕が手元で上記の例にもう 1 個 `return { hoge: true }` のようなステートメントを足したらあっさりバラけたので、どこまで Type Alias が保存されるかは不明です。

Type Alias 自体、 "Alias" というくらいに、飽くまで型に別名を付けて使い回す程度の機能であったため、tooltip やエラーメッセージの読みやすさの観点でいうと微妙なところがある、というのは知っておいて欲しいポイント。

### Stricter Checks For The `in` Operator

```ts
"foo" in 42;
```

のように、Object type でないものに `in` 演算子を使ったときにコンパイルエラーとしてれる、というもの。

正確に言うと、4.1 以前でも上記の例では `TS2361` のエラーになっていたが、このエラーチェックがより厳密になった模様。具体的にどういう条件のときの挙動が変わったかは、 https://github.com/microsoft/TypeScript/pull/41928/filesを見に行かないと分から無さそう（倉見はそこまで見れなかったです）。

### `--noPropertyAccessFromIndexSignature`

「Index 形式で定義されたプロパティについて、`opts.xxxx` のようなドット形式でのアクセスを許さず、 `opts['xxxx']` のような Index 形式でのアクセスのみを許容するようにする」というオプションです。

以下の `Options` のような、Index Signature と Property Signature が混在した型が出てくるときに目立った差が現れます。

```ts
interface Options {
  /** File patterns to be excluded. */
  exclude?: string[];

  /**
   * It handles any extra properties that we haven't declared as type 'any'.
   */
  [x: string]: any;
}

function processOptions(opts: Options) {
  // Notice we're *intentionally* accessing `excludes`, not `exclude`
  if (opts.excludes) {
    console.error(
      "The option `excludes` is not valid. Did you mean `exclude`?"
    );
  }
}
```

従来の TS では上記のコードはエラーになりませんでしたが、 `--noPropertyAccessFromIndexSignature` を有効にしている場合「`opts.exclude` は許容するが `opts.exlucdes` は許容しない」という挙動になります。

4.1 で導入された `--noUncheckedIndexedAccess` というので「未検査の Index Signature Access は Optional 扱い（undefined と和を取られる）」というオプションが導入されましたが、これをさらに堅く運用したい場合、今回導入された `--noPropertyAccessFromIndexSignature` も併用すると良さそうです。

`--strict` ファミリーではないので、明示的に tsconfig で有効化する必要がある機能です。

### `abstract` Construct Signatures

下記のように、Constructor Type に `abstract` modifier が追加されました。

```ts
type AbstractConstructor<T> = abstract new (...args: any[]) => T
```

`abstract` の利用例として、「抽象クラスに関数を mixin する」というユースケースが紹介されています（個人的にそこまで abstract class をガリガリ使うことがないので、あまり有難味がわかってない）。

```ts
abstract class SuperClass {
  abstract someMethod(): void;
  badda() {}
}

type AbstractConstructor<T> = abstract new (...args: any[]) => T;

function withStyles<T extends AbstractConstructor<object>>(Ctor: T) {
  abstract class StyledClass extends Ctor {
    getStyles() {
      // ...
    }
  }
  return StyledClass;
}

class SubClass extends withStyles(SuperClass) {
  someMethod() {
    this.someMethod();
  }
}
```

### Understanding Your Project Structure With `--explainFiles`

「TypeScript コンパイラがどのファイルを読み込んだか」を出力してくれるオプションです。従来の `--traceResolution` （モジュールの解決過程を verbose 出力する）に似ていますが、暗黙的に読み込まれる `lib.d.ts` なども含めて出力してくれるところなどが異なります。

### Improved Uncalled Function Checks in Logical Expressions

`&&` や `||` のオペランドで「関数呼び出しを意図していそうなのに関数自体を渡してしまっている」ケースをエラーとしてくれるようになりました。

```ts
function shouldDisplayElement(element: Element) {
  // ...
  return true;
}

function getVisibleItems(elements: Element[]) {
  return elements.filter(e => shouldDisplayElement && e.children.length);
  //                          ~~~~~~~~~~~~~~~~~~~~
  // This condition will always return true since the function is always defined.
  // Did you mean to call it instead.
}
```

### Relaxed Rules Between Optional Properties and String Index Signatures

タイトルのママですが、オプショナルなプロパティを Index 記法の型に代入する際のチェックルールが緩和されました。

4.1 では、以下の代入はエラーとなっていました。右辺となっている型の値部分 `number | undefined` は左辺型の値分である `number` のサブタイプでは無いからです。

```ts
type WesAndersonWatchCount = {
  "Fantastic Mr. Fox"?: number;
  "The Royal Tenenbaums"?: number;
  "Moonrise Kingdom"?: number;
  "The Grand Budapest Hotel"?: number;
};

declare const wesAndersonWatchCount: WesAndersonWatchCount;
const movieWatchCount: { [key: string]: number } = wesAndersonWatchCount;
//    ~~~~~~~~~~~~~~~ error!
// Type 'WesAndersonWatchCount' is not assignable to type '{ [key: string]: number; }'.
//    Property '"Fantastic Mr. Fox"' is incompatible with index signature.
//      Type 'number | undefined' is not assignable to type 'number'.
//        Type 'undefined' is not assignable to type 'number'. (2322)
```

4.2 では上記コードのような assignment を行ってもエラーにはなりません。`WesAndersonWatchCount` はすべてオプショナルなプロパティしか持たないので、直感的な挙動に近づいたと言ってよいでしょう。

明示的な `undefined` との和を取っている場合、従来通りエラーとなります。飽くまで `?:` による表現に限ります。

```ts
type WesAndersonWatchCount = {
  "Fantastic Mr. Fox": number | undefined;
};
```

## Breaking Changes

- `Intl` , `ResizeObserver` 関連で lib.d.ts が変更されています
- Generator Function での `--noImplicitAny` の挙動が変更されています
- `--strictNullChecks` が有効な場合に、 `&&` や `||` 演算子の付近でエラーチェックが厳密になっているため、既存のコードについて新しいエラーが発生する可能性があります
- `in` 演算子のチェックが厳密からされているため、既存のコードについて新しいエラーが発生する可能性があります
- タプル型の最大長（tsc が内部で利用するパラメータ）に変更が入りました。複雑な型パズルのようなことをしていると、`Expression produces a tuple type that is too large to represent.` というエラーが発生する可能性があります。
- `import "hoge.d.ts"` のように `d.ts` を明示的に指定した import ができなくなります。 `import "hoge"` のように変更してください。
