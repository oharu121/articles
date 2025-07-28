# prevって何？なんで直接setCount(count + 1)じゃダメなの？

# はじめに

Reactでカウンターなどの状態を管理していると、こんなコードを見かけることがあります。

```tsx
setCount((prev) => prev + 1);
```

「`prev`って何？」「なんで直接`setCount(count + 1)`じゃダメなの？」と疑問に思った方も多いのではないでしょうか。

この記事では、**`prev`の正体とその重要性**を、シンプルなカウンターの例を使ってわかりやすく解説します！

# `prev`の意味とは？

`prev`とは、**「前の状態（previous state）」**のことです。

Reactの`useState`で定義した状態更新関数（例：`setCount`）には、**関数を引数として渡すことができます**。

```tsx
const [count, setCount] = useState(0);

// 関数を渡す方法
setCount((prev) => prev + 1);
```

このとき、Reactは自動的に関数の引数に**最新の状態**（= `prev`）を渡してくれるのです。

# なぜ直接`setCount(count + 1)`ではダメなの？

複数回連続で状態を更新する場合、**古い状態に基づいてしまうことがある**からです。

❌ NGな例

```tsx
const handleClick = () => {
  setCount(count + 1)
  setCount(count + 1)
}
```
この関数が呼ばれた瞬間、**今の`count`の値が確定しており、それ以降は変わらない**のです。

* `count`はこの関数内では「ただの変数」として扱われる
* `setCount(count + 1)`を2回呼んでも、どちらも「古いcount + 1」で計算される
* それぞれの`setCount`は同じ値を次の状態としてReactに渡す

```tsx
const handleClick = () => {
  // コールされるときcount = 1の場合
  setCount(0 + 1)
  setCount(0 + 1)
}
```

✅ OKな例（`prev`を使う）

```tsx
const handleClick = () => {
  setCount((prev) => prev + 1);
  setCount((prev) => prev + 1);
}
```

この書き方なら、Reactは1回目の更新後の最新の値を`prev`に渡してくれるので、**正しく2回加算されます（+2）**。

👀 `prev`を使うべきパターン

| 状況           | 書き方                            | 説明                    |
| ------------ | ------------------------------ | --------------------- |
| 1回だけ更新       | `setCount(count + 1)`          | 状態に依存していないならOK        |
| 状態に依存して複数回更新 | `setCount((prev) => prev + 1)` | **このパターンで`prev`が必要！** |

# まとめ

* `prev`は **「直前の状態」** を意味する
* `setState((prev) => …)` という形で使う
* 特に、**状態に依存した更新を複数回行うときに必須**
* Reactが **最新の状態を保証して渡してくれる** ので安心！

✍️ おまけ：サンプルコード全体

```tsx
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);

  const handleDoubleIncrement = () => {
    setCount((prev) => prev + 1);
    setCount((prev) => prev + 1);
  };

  return (
    <div>
      <p>カウント: {count}</p>
      <button onClick={handleDoubleIncrement}>+2する</button>
    </div>
  );
}
```

このように、`prev`を使えば\*\*「意図した通りの更新」\*\*を安全に行うことができます。Reactで状態管理を行う上でとても大事なポイントなので、ぜひ覚えておきましょう！
