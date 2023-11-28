# ng-japan OnAir #52 ゆるトーク「Angular と Storybook」

## Storybook x Angular で興味がある部分

Quramy 自身が storybook/angular に Contribute していた頃は、Storybook の webpack 設定と Angular CLI の webpack 設定をゴリッと merge したりするような部分が大変だった。今はどうなっているんだろう。

https://speakerdeck.com/quramy/storybook-and-angularcli

## VRT with Storybook

https://speakerdeck.com/quramy/screenshot-testing-with-angular

Angular を使うかどうかと関係なく、Frontend には有効だと思っている。

E2E と比較すると、Storybook(= Component Catalogue) のメンテナンスは苦になりにくいと信じている。もちろん E2E や Integrated Test じゃないと担保できないレベルのテストは無くならないが「特定の条件下で CSS が意図通りか」を主軸に置くのであれば、E2E はコスパが悪い。

## Storybook を運用する上での工夫とか

なんかある？

Quramy は「Component に対する単体テストコードだと思って書け」と言うようにしている。

「初見者でも参入しやすい」 のような観点については https://storybook.js.org/blog/structuring-your-storybook/ にノウハウが記載されているので、プロジェクト導入時に参考にすると良さそう。

## 最近の Storybook の動向について

### CSF 3.0

https://storybook.js.org/blog/component-story-format-3-0/

書き味自体もよくなったと思う。特に React の場合だと、CSF2 で以下のように書くと TypeScript 上エラーになるし、そもそも「Arrow Function にパラメータ付与する」というのがあまり気持ちよくないし。

```tsx
// CSF 2.0
export const Default = args => <Button {...args} />;

// ↓のコードだけだと Compile Error
Default.parameters = {};
```

### Interaction(Play Function)

今までの Storybook では、「描画したい状態」を外から与えるしかできなかった。
( property や service stub 差し替え)

Play function はこれらに加えて「UI の操作」を事前状況に含めることができるようになる。

事前条件に利用可能な手法が増えるということは、Storybook が表現できるテストケースの幅が広がった、ということ。

たとえば「フォーカス時にフォーカスリングが表示されること」のようなパターンも自動テストとして記述できるようになった。

## testing-angular

https://storybook.js.org/addons/@storybook/testing-react を使うと、CSF を React のテストコードから再利用できるようになる。

これの Angular 版があれば喜ばれるのでは？

## References

- [Storybook and AngularCLI (めっちゃ古い)](https://speakerdeck.com/quramy/storybook-and-angularcli)
- [Screenshot testing with Angular](https://speakerdeck.com/quramy/screenshot-testing-with-angular)
- [Structuring your Storybook](https://storybook.js.org/blog/structuring-your-storybook/)
- [Component Story Format 3.0](https://storybook.js.org/blog/component-story-format-3-0/) - [Storybook の play function と VRT](https://qiita.com/Quramy/items/d332e37dded505daac90)
- [Interaction Testing with Storybook](https://storybook.js.org/blog/interaction-testing-with-storybook/)
