# xUnit Patters 読書メモ

## xUnit Patterns

- Code Smells
  - Obscure Test
  - Conditional Test Logic
  - Hard-to-Test Code
  - Test Code Duplication
  - Test Logic in Production
- Behavior Smells
  - Assertion Roulette
  - Erratic Test
  - Fragile Test
  - Frequent Debugging
  - Manual Intervention
  - Slow Tests
- Project Smells
  - Buggy Tests
  - Developers Not Writing Tests
  - High Test Maintenance Cost
  - Production Bugs

## 見出しから想像する内容

- Code Smells: コードから見て取れる悪臭
  - Obscure Test: あいまいなテスト. 何がしたいのかわからない、テストコードが存在する価値が判断できない?
  - Conditional Test Logic: テストコード中に if 文のような (複雑な) 条件が出てきてしまい、テストの妥当性が危うくなっていく、ということかな？
  - Hard-to-Test Code: プロダクションコード側のテスタビリティが低くいままの状態で無理やりテストコードを書いてしまった、とか？
  - Test Code Duplication: テストコード自体が重複(コピペの嵐) になってしまっていて、テスト対象を変更したときに修正しなければいけない箇所が多くて悲しい、とか？
