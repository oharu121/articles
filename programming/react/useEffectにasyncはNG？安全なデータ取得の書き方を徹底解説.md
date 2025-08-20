## useEffect に async は NG？安全なデータ取得の書き方を徹底解説

## はじめに

React でデータフェッチを行うとき、`useEffect`と`async/await`を組み合わせるのはよくあるパターンです。

しかし、次のように書いたことはありませんか？

```tsx
React.useEffect(async () => {
  const data = await fetchData({ query, page, tag });
  setResults(data.results);
}, [query, page, tag]);
```

実はこれ、**非推奨の書き方**です。

## なぜ useEffect(async () => {}) がダメなのか？

`useEffect`に渡す関数は、React の仕様上、

- `undefined` を返す
- または「クリーンアップ関数」を返す必要があります。

ところが、`async`関数は常に **Promise** を返すため、**React が期待している戻り値の形式に合いません**。

その結果…

- クリーンアップが正しく動作しない
- 将来の React のアップデートで警告やバグの原因になる

といった問題が起こる可能性があります。

**✅ 正しい書き方：async 関数を定義する**

```tsx
React.useEffect(() => {
  const handleFetchData = async () => {
    setLoading(true);
    const data = await fetchData({ query, page, tag });
    setResults(data.results);
    setLoading(false);
  };

  handleFetchData();
}, [query, page, tag]);
```

このように、**非同期関数は useEffect の中で定義して実行する**のが基本です。

## async 関数を useEffect の外に出すべき？

一見、`handleFetchData` を外に出したほうが「きれい」になりそうに見えます。でも、それには注意が必要です。

**❌ 原則：依存変数がある場合は外に出さないほうが良い**

```tsx
const handleFetchData = async () => {
  const data = await fetchData({ query, page, tag }); // ❌ query などが stale になる可能性
  setResults(data.results);
};

useEffect(() => {
  handleFetchData(); // ❌ 依存配列だけ更新されても中身が古いまま
}, [query, page, tag]);
```

このコードは、`handleFetchData` が `query`, `page`, `tag` をクロージャとして**キャプチャしてしまう**ため、値が古くなる可能性があります。

**✅ 例外：再利用したいときは useCallback + 依存配列で管理**

たとえば「再取得」ボタンを用意したい場合：

```tsx
const handleFetchData = useCallback(async () => {
  setLoading(true);
  const data = await fetchData({ query, page, tag });
  setResults(data.results);
  setLoading(false);
}, [query, page, tag]);

useEffect(() => {
  handleFetchData();
}, [handleFetchData]);

return <button onClick={handleFetchData}>再取得</button>;
```

これは、

> **handleFetchData が変わったときだけ、呼び出す**

という意味の `useEffect` です。**分解してみると：**

> 「依存の値 ( `query`, `page`, `tag` ) が変わった → 関数 ( `handleFetchData` ) が変わった → 副作用 ( `useEffect` ) 実行」という**意図した挙動**を実現

1. `handleFetchData()` は API を呼ぶなどの副作用（＝副作用関数）です。
2. `useEffect` の依存配列 `[handleFetchData]` に注目。

   - これは「`handleFetchData` の**中身が変わったとき**に再実行する」設定です。
   - 中身が変わるというのは、`useCallback` で依存が変わって再生成されたときのこと。

**メリット**

- `useCallback` を使えば関数の再生成を防げる
- `query`, `page`, `tag` を依存に入れることで**常に最新値が使われる**

## まとめ

- `useEffect`には直接 `async` 関数を渡さない！
- `handleFetchData` は中で定義 or `useCallback`で管理
- React の再レンダーや依存関係を理解したうえで、**関数のスコープ設計**を考えるのが大事

| シチュエーション            | 推奨されるパターン                |
| --------------------------- | --------------------------------- |
| useEffect 内だけで使う      | `useEffect`の中で定義             |
| 他のイベントでも使いたい    | `useCallback`でメモ化して外に出す |
| 依存がない / グローバル関数 | 外に出しても OK                   |
