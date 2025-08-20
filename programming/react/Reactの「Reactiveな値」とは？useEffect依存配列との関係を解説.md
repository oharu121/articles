## React の「Reactive な値」とは？useEffect 依存配列との関係を解説

## はじめに

React で`useEffect`を書くとき、よく悩むのがこれ：

> 「この値、依存配列に入れたほうがいいの？入れなくていいの？」

実は、すべての判断基準は **"Reactive な値かどうか"** に集約されます。

## そもそも「Reactive な値」とは？

> **レンダーのたびに変わる可能性がある値のこと。**

React は**その値が変わったときだけ** `useEffect` を再実行したいので、Reactive な値だけを依存配列に入れる必要があります。

| 種類                 | 例                          | Reactive?                 |
| -------------------- | --------------------------- | ------------------------- |
| `props`              | `props.name`                | ✅ Yes                    |
| `state`              | `count`, `user`             | ✅ Yes                    |
| コンポーネント内変数 | `const foo = something()`   | ✅ Yes                    |
| `setXxx`関数         | `setCount()`                | ❌ No（参照が変わらない） |
| `localStorage`       | `localStorage.getItem()`    | ❌ No                     |
| グローバル関数       | `Number()`, `Math.random()` | ❌ No                     |

## 1. 依存配列にいれるべきパータン

**❌ ダメな例：名前が変わっても反応しない**

```tsx
function Greeting({ name }: { name: string }) {
  useEffect(() => {
    console.log(`Hello, ${name}`);
  }, []); // ⛔ nameを無視している！

  return <div>Hello!</div>;
}
```

このままだと、`name` が変わってもログは一度しか表示されません。**理由は、`name` が Reactive な値なのに、依存配列に含めてないから**。

**✅ 正しい例**

```tsx
useEffect(() => {
  console.log(`Hello, ${name}`);
}, [name]); // ✅ Reactiveな値は依存配列に入れる
```

## 2. 依存配列に入れなくていいパータン

```tsx
useEffect(() => {
  const item = localStorage.getItem("index");
  if (item) setIndex(Number(item));
}, []); // ⭕ OK
```

なぜこれで OK？

- `localStorage.getItem` は変化しない
- `Number()` は純粋関数
- `setIndex` は React が毎回同じものを返す保証あり

→ つまり **非 Reactive な値なので、依存配列に入れなくて OK**。

## 判断に迷ったときのチェックリスト

> **この値は…**
>
> - [ ] props で受け取っている？
> - [ ] useState で定義された state？
> - [ ] 関数コンポーネントの中で定義されてる？
>
> → ✅ YES なら依存配列に入れよう！

**🎁 おまけ：useEffect 依存配列のバグを防ぐ開発 Tips**

- `eslint-plugin-react-hooks` を入れると、依存忘れを自動検出してくれます

  ```bash
  npm install eslint-plugin-react-hooks --save-dev
  ```

## おわりに

「依存配列、何を入れたらいいか迷う…」という方へ、

👉 **「それ、Reactive な値か？」で判断しましょう。**

「依存配列の最適化」は**React 開発のバグの温床**なので、最初にマスターすべきスキルです！
この記事があなたの `useEffect` バグ撲滅の一助になれば幸いです！
