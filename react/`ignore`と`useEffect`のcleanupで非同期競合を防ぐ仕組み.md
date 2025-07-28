# `ignore`と`useEffect`のcleanupで非同期競合を防ぐ仕組み

# はじめに

非同期処理を`useEffect`内で書いていると、こんな問題に遭遇したことはありませんか？

> 古いリクエストのレスポンスが後から返ってきて、UIが古い状態で上書きされてしまう

このような **非同期の競合（race condition）** は、実は `useEffect` を使ったネットワーク通信でよく起きる落とし穴です。

この記事では、この問題を解決するための超実用的テクニックである `ignore` フラグと `useEffect` の cleanup 関数の仕組みを徹底解説します。

# 問題

> ステートが「古いレスポンス」で上書きされる

以下はポケモンAPIを使ったカルーセルの一部実装例です：

```tsx
useEffect(() => {
  async function fetchData() {
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
    const data = await res.json();
    setPokemon(data); // ← ここで古いデータが上書きされる可能性あり
  }
  fetchData();
}, [id]);
```

一見よさそうですが、例えば次のような状況を想像してみてください：

1. id=1 の時に `fetch` 開始（fetching中）
2. **すぐ**に「次へ」ボタンを押して、APIを `id=2` に変更
3. `id=2` のリクエストが先に終わり、`id=2` の結果を表示
4. しかしその後に `id=1` のレスポンスが帰ってきて**上書き**されてしまう

🤔結果：UIが古いポケモンを表示してしまう

#  解決法

> `ignore` フラグで古いリクエストを無視する

Reactの `useEffect` には「クリーンアップ関数」という機能があります。これは、**次の `useEffect` が走る直前**や**コンポーネントがアンマウントされる直前**に呼ばれます。

この性質を利用して、こうします：

```tsx
useEffect(() => {
  let ignore = false;

  async function fetchData() {
    setLoading(true);
    setError(null);

    const { response, error } = await fetchPokemon(id);

    if (!ignore) {
      if (error) setError(error);
      else setPokemon(response);
      setLoading(false);
    }
  }

  fetchData();

  return () => {
    ignore = true;
  };
}, [id]);
```

**💡 解説**

✅ 1. `ignore` はクロージャで閉じている

>* `ignore` フラグは **この `useEffect` がまだ有効かどうか** を表す
>* `useEffect` 内の `ignore` はその時点でのスコープに閉じている
>* 後から別の `useEffect` が実行されても、**前の `ignore` がtrueになるだけ**で、それに依存した処理は止まる

✅ 2. Reactは`useEffect`を順番通りに実行＆クリーンアップ

* 次の `useEffect`（`id = 2`） が走ると、前の `useEffect`（`id = 1`）は`ignore = true` になる
* そのため、**後から返ってきた古いレスポンス**は `ignore === true` によって無視される

✅ 3. 状態更新は最新のリクエストだけが行う

* 副作用が発生する前に `ignore` をチェックすることで、UIが誤って古い状態に戻ることを防げます。
* つまり、**古いリクエストのレスポンスを無視できる**

**📌 なぜキャンセルではなく「無視」なのか？**

`fetch` は中断（abort）できますが、より簡単に安全性を保ちたい場合、**「レスポンスを無視する」という戦略のほうが実装が楽**です。

* キャンセルAPI（`AbortController`）を使うと少し複雑
* `ignore` フラグは **副作用の発生を防ぐ** だけなので手軽

**📄 `ignore` パターンのテンプレート**

```tsx
useEffect(() => {
  let ignore = false;

  async function fetchData() {
    const result = await someAsyncFunction();
    if (!ignore) {
      // state更新など
    }
  }

  fetchData();

  return () => {
    ignore = true;
  };
}, [dependencies]);
```

これはAPI通信以外でも、**非同期処理の副作用がある全てのケースに使えます**。

# **cleanup function の正体とは？**

`useEffect` に return された関数のこと。
React はこれを **次の effect 実行前** または **コンポーネントがアンマウントされる直前** に実行します。

```tsx
useEffect(() => {
  // 副作用の実行
  startSubscription();

  // 👇 これが cleanup function
  return () => {
    stopSubscription(); // 後始末
  };
}, [deps]);
```

**🕒 React内部での処理タイミング**

React の実行タイミングは以下の通り：

1. 初回レンダリング → effect 実行
2. `deps` の変更 → **前の cleanup → 新しい effect 実行**
3. アンマウント時 → 最後の cleanup 実行

**🧪 どんなときに使うの？**

| 使用ケース                        | cleanup の役割                      |
| ---------------------------- | -------------------------------- |
| イベントリスナーの登録                  | 削除する（removeEventListener）        |
| WebSocket の接続                | 切断する（socket.close()）             |
| `setInterval` / `setTimeout` | `clearInterval` / `clearTimeout` |
| fetch の競合防止                  | `ignore = true` のようなフラグセット       |
| サブスクリプション                    | 解除処理を呼ぶ                          |

# まとめ

| 問題                  | 解決                        |
| ------------------- | ------------------------- |
| 非同期レスポンスが競合してUIが壊れる | `ignore` フラグで古いリクエストを無視する |

* 非同期処理は「競合」によって意図しないUI更新を起こすことがある
* `ignore` は\*\*現在の effect が「まだ有効か」\*\*を判断するためのフラグ
* `useEffect` の**クリーンアップ関数内で `ignore = true` にする**だけでOK
* `ignore` フラグと `useEffect` のクリーンアップを組み合わせれば安全に防げる
* クロージャ＋Reactの実行順で**古いリクエストを安全に無視**できる
* 複雑なキャンセル処理なしで **副作用の安全性**を確保できる

非同期の副作用に悩んでいる方は、ぜひこのパターンを取り入れてみてください！
