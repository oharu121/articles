# はじめに

Promise.allは、複数のPromiseオブジェクトを並行して処理するための非常に便利なJavaScriptの機能です。複数の非同期操作を完了する必要があり、これらの操作が相互に依存していない場合に、Promise.allは特に便利です。

# Promise.allを利用前

以下の関数を参考にしてください。この関数では、すべてのステップが前のステップが終了するのを待ってから実行されますが、実際にはご飯を炊くことと野菜を切ることは同時に行うことができます。そのため、かなりの時間が無駄になっています。

```typescript:example.ts
// チャーハンを作る関数
const makeDish = async (): Promise<void> => {
  await prepareIngredients(); //食材を用意する
  await cookRice(); //ご飯を炊く
  await chopVegetables(); //野菜を切る
  await addIngredients(); //食材を入れる
  await stirFry(); //炒める
};
```

# Promise.allを利用後

Promise.all を使用して、ご飯を炊く作業と野菜を切る作業を同時に行うことで、上述の問題を解決することができます。これにより、これらのタスクが互いに依存していないため、同時に実行することで全体の処理時間を短縮できます。以下のコードは、このアプローチを実装したものです。

```typescript:example.ts
// チャーハンを作る関数
const makeDish = async (): Promise<void> => {
  await prepareIngredients(); // 食材を用意する
  // ご飯を炊くと野菜を切る作業をPromise.allで同時に実行
  await Promise.all([cookRice(), chopVegetables()]);
  await addIngredients(); // 食材を加える
  await stirFry(); // 炒める
};
```

# まとめ

Promise.all を使用することで、複数の非同期操作を並行して実行し、全体の処理時間を効率的に短縮することができます。この方法は、特に互いに依存しないタスクを同時に実行する必要がある場合に有効です。
