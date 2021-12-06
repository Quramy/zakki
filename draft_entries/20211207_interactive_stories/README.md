# Storybook の play function と VRT

## play function is 何

Storybook 6.4 では、CSF 3.0 や new Story Store など、いくつかの新機能が導入されましたが、その目玉機能の 1 つが `play` function です。

```tsx
// RegistrationForm.stories.ts|tsx

import React from "react";

import { ComponentStory, ComponentMeta } from "@storybook/react";

import { screen, userEvent } from "@storybook/testing-library";

import { RegistrationForm } from "./RegistrationForm";

export default {
  /* 👇 The title prop is optional.
   * See https://storybook.js.org/docs/react/configure/overview#configure-story-loading
   * to learn how to generate automatic titles
   */
  title: "RegistrationForm",
  component: RegistrationForm
} as ComponentMeta<typeof RegistrationForm>;

const Template: ComponentStory<typeof RegistrationForm> = args => (
  <RegistrationForm {...args} />
);

export const FilledForm = Template.bind({});
FilledForm.play = async () => {
  const emailInput = screen.getByLabelText("email", {
    selector: "input"
  });

  await userEvent.type(emailInput, "example-email@email.com", {
    delay: 100
  });

  const passwordInput = screen.getByLabelText("password", {
    selector: "input"
  });

  await userEvent.type(passwordInput, "ExamplePassword", {
    delay: 100
  });
  // See https://storybook.js.org/docs/react/essentials/actions#automatically-matching-args to learn how to setup logging in the Actions panel
  const submitButton = screen.getByRole("button");

  await userEvent.click(submitButton);
};
```

上記のコードは [Storybook の公式ドキュメント](https://storybook.js.org/docs/react/writing-stories/play-function#writing-stories-with-the-play-function)から拝借してきています。

今までの Storybook は、与えられたプロパティに従ってコンポーネントを描画するだけでしたが、 **play function によってユーザーインタラクション発生後のコンポーネントの状態** まで含めて描画できるようになったわけですね。

ちなみに上記コード例は CSF 2.0 形式で書かれていますが、今から新しく Story を書き起こすのであれば CSF 3.0 で書いたほうがいいでしょう。CSF 2.0 / 3.0 の記載方法の違いについては、以下のエントリを参考にしてください。

https://qiita.com/p_irisawa/items/3fab9b6a961503b4793b

## play function と Storycap

さて、play function を含んだ Story に対して VRT (Visual Regression Testing) を実行するのも、さして難しくありません。

手前味噌ですが、僕が課外活動で公開している OSS の https://github.com/reg-viz/storycap を使ってもらえれば、特に新しいことを意識しなくても「play function を動作させた結果」のキャプチャを取得できます。

Storycap 側では、現状は特に play function に対する対応を行っていないのですが、やってみたら普通に動いてしまって当の開発者も割と驚いています。

Storycap は VRT での利用という特性上、同じソースコードからは同じキャプチャが生成するために「レンダリングエンジンが忙しいかどうか」を検査するようにしています。以下のメトリクスを監視しておき「一定のフレームの間でこれらが同じに値に収束すること」をキャプチャ実行の必要条件にしているのです。

- DOM Node 数
- Style Calculation の実行回数
- Layout Calculation の実行回数

元々は React や Angular といった UI Framework に依存させずにキャプチャをトリガーするための仕組みとして導入していたのですが、この監視機構が play function に対しても動作するため、play function で DOM をガチャガチャいじっているうちはキャプチャが遅延され、操作が一段落したらキャプチャ実行されるようになっています。

逆に、以下のように、play function 内で 中途半端に delay が発生すると、現状の Storycap は「レンダリングエンジンが安定した」と誤解してキャプチャを実行してしまうかも。

```ts
await userEvent.type(passwordInput, "ExamplePassword", {
  delay: 100
});
```

Storycap を実際に VRT に組み込むワークフローについては、既にメンバーにかかれてしまったため、そちらに譲ることにします。

https://qiita.com/sakamuuy/items/44a109532f06e0b619e3

## 実際に play function を試した所感

実際、 Storybook 6.4 がリリースされた辺りから play function を実業務の案件で利用するようにしています。

Storycap や reg-suit など、VRT の基盤に手を入れずに導入できたのも楽で良かったのですが、効果として大きいなと感じたのは「form 周りのインタラクションテストをちゃんと書けるようになった」という点でしょう。

特に、最近担当している案件では react-hook-form を採用する機会が多いです。react-hook-form は Uncontrolled な Form、すなわち 各 form control の値管理は DOM に委ねる思想のライブラリです。Controlled なフォームに比べるとパフォーマンス上の利点もありますし、コードの書き味も悪くないのですが、フォームの状態と DOM の状態の結合度が高くなり、Storybook での確認という観点では若干の不満がありました。

例として、react-hook-form を利用したログインフォームを考えてみたいと思います。

```tsx
import { useForm } from "react-hook-form";

export type LoginForm = {
  readonly name: string;
  readonly password: string;
};

export function MyForm({
  onSubmit
}: {
  readonly onSubmit: (formValue: LoginForm) => void;
}) {
  const {
    register,
    formState: { isValid, errors },
    handleSubmit
  } = useForm<LoginForm>({ mode: "onBlur" });
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <label>
        Name
        <input type="text" {...register("name", { required: true })} />
      </label>
      {errors.name && errors.name.type === "required" && (
        <span>This is required</span>
      )}
      <label>
        Password
        <input type="password" {...register("password", { required: true })} />
      </label>
      {errors.password && errors.password.type === "required" && (
        <span>This is required</span>
      )}
      <button type="submit" disabled={!isValid}>
        Sign in
      </button>
    </form>
  );
}
```

上記の `MyForm` コンポーネントについて、外部から受けとる prop は `onSubmit` だけであるため、今までの Storybook で描画できるのは初期状態（空フォーム）だけでした。

描画された Story をブラウザから操作すれば、バリデーションや活性制御の動きが実際に確認できるものの、それは開発者が実際に操作しなくてはいけません。

自動テストとしてバリデーションエラーが描画された状態を作り出すためには、Storybook のためだけに `formState` を prop とする別の Component に切り出すなりをしないといけないのですが、まぁ割と面倒ですし、「Storybook の prop として与えた `formState` が本当に UI から到達可能な state なのか」という検証は別に必要になります。

以下のような play function を持った Story を用意しておくことで、バリデーションエラーが発生する状態を再現できますし、その際の「エラーメッセージが表示されること」や「送信ボタンが非活性であること」はについては、VRT にまかせておけばアサーションの記述に頭を悩ますこともありません。

```ts
export const Validation: ComponentStoryObj<typeof MyForm> = {
  play: async ({ canvasElement }) => {
    const screen = within(canvasElement);
    await userEvent.click(screen.getByLabelText("Name"));
    await userEvent.tab();
  }
};
```

フォーム以外にも「クリックしたら詳細部分が展開されるアコーディオン UI」「ホバーしたら表示されるツールチップ UI」など、ちょっとした操作で UI の状態が変更されるようなコンポーネントも play function を追加していくことで、自動テストを拡充できるため、ガシガシと活用していきたいですね。
