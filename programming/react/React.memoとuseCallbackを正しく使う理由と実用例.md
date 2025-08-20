## React.memo と useCallback を正しく使う理由と実用例

## はじめに

React アプリのパフォーマンスやバグの原因を調べていると、よく出てくるのが `React.memo` や `useCallback`。しかし「**再レンダリングを防ぐため**」というだけの理解では、十分に使いこなすことはできません。

この記事では、以下のような疑問を実用コード付きで解説します：

- `React.memo`はどんなときに使うべき？
- `useCallback`は何のためにある？
- **stale closure（古い状態を保持してしまうバグ）** をどう防ぐ？
- 実際にどう組み合わせれば効果的？

**🎯 ゴール**

- `React.memo`と`useCallback`の使い分けを理解する
- 子コンポーネントの**無駄な再レンダリング防止**
- **stale closure 回避**と**props の安定性**を両立させる

## なぜ useCallback と memo を併用するのか？

✅ 1. React.memo の目的：再レンダリング最適化

- 親が再レンダリングされても、**子に渡す props が変わってなければ描画しない**
- ただし、関数やオブジェクトが**毎回新しい参照になっていると無意味**

```tsx:TodoItem.tsx
import React from "react";

interface TodoItemProps {
  item: string;
  onRemove: () => void;
}

// React.memoでラップ → propsが変わらない限り再描画されない
export const TodoItem = React.memo(({ item, onRemove }: TodoItemProps) => {
  console.log(`Rendering: ${item}`);
  return (
    <div className="flex justify-between border p-2 my-1">
      <span>{item}</span>
      <button onClick={onRemove} className="text-red-500">Delete</button>
    </div>
  );
});
```

✅ 2. useCallback の役割：関数の参照を安定化

- `useCallback(() => ..., [依存])` により関数を再生成しない
- props として渡すときや、useEffect の依存に使うときに便利

```tsx:TodoList.tsx
import React, { useState, useCallback } from "react";
import { TodoItem } from "./TodoItem";

export const TodoList: React.FC = () => {
  const [items, setItems] = useState(["Learn React", "Write Blog"]);
  const [input, setInput] = useState("");

  const handleAdd = () => {
    if (input.trim() !== "") {
      setItems((prev) => [...prev, input.trim()]);
      setInput("");
    }
  };

  // useCallbackで関数をメモ化（毎回新しい参照にならないように）
  const handleRemove = useCallback(
    (index: number) => {
      setItems((prev) => prev.filter((_, i) => i !== index));
    },
    []
  );

  return (
    <div className="max-w-md mx-auto mt-10 p-4 border rounded">
      <h1 className="text-xl font-bold mb-4">ToDo List</h1>
      <div className="flex gap-2 mb-4">
        <input
          className="border p-1 flex-1"
          value={input}
          onChange={(e) => setInput(e.target.value)}
        />
        <button className="bg-blue-500 text-white px-3" onClick={handleAdd}>
          Add
        </button>
      </div>
      {items.map((item, idx) => (
        <TodoItem key={idx} item={item} onRemove={() => handleRemove(idx)} />
      ))}
    </div>
  );
};
```

## stale closure（古い状態のバグ）とは？

```tsx
const handleClick = () => {
  console.log(count); // 古いcountのまま
};
```

この関数が生成された時点の`count`を保持してしまうことで、**最新の状態にアクセスできないバグ**が発生します。

✅ 解決法（useCallback）

```tsx
const handleClick = useCallback(() => {
  console.log(count); // 最新のcountを依存配列で保障
}, [count]);
```

| 技術          | 効果                                     |
| ------------- | ---------------------------------------- |
| `React.memo`  | 子コンポーネントの再描画を防ぐ           |
| `useCallback` | 関数の参照を固定し、`React.memo`と相性 ◎ |

## 補足：`useEffect` vs `useCallback`

useCallback と useEffect、どちらも依存配列で stale closure を防げる。でもどう違うの？

🤔 たしかに、

> `useCallback` や `useEffect` の依存配列に state を正しく入れることで、`stale closure`を解決できます。

と言われると、こんな疑問がわくはずです：

> 「じゃあ `useEffect` だけでいいんじゃないの？
> わざわざ `useCallback` って使う意味あるの？」

**✅ 結論**

| フック        | 目的                                                 |
| ------------- | ---------------------------------------------------- |
| `useEffect`   | **副作用を実行する**（データ取得・イベント登録など） |
| `useCallback` | **関数の再生成を防ぐ**（props 最適化やメモ化のため） |

**🧪 useEffect：関数を実行したいとき**

```tsx
useEffect(() => {
  console.log("countが変わったよ:", count);
}, [count]);
```

- `count`が変わったときに副作用（console.log など）を**実行するためのフック**

**🔁 useCallback：関数を渡したいとき**

```tsx
const handleClick = useCallback(() => {
  console.log("count:", count);
}, [count]);
```

- この関数を**子コンポーネントに props で渡す**
- `React.memo`と一緒に使えば、**不要な再レンダリングを防げる**

**✍️ 実務的な判断ポイント**

- **イベントハンドラをコンポーネントに渡すとき → useCallback**
- **API 通信、監視、DOM 更新など → useEffect**
- **React.memo + props 最適化したい → useCallback 必須**

## おわりに

`React.memo`は「パフォーマンスチューニング」のためのツールですが、
`useCallback`と併用することで、**state の一貫性やバグの予防**にも役立ちます。

もし、複雑なアプリの中で子コンポーネントの**再描画が多い、パフォーマンスが気になる、意図しない挙動が出ている**場合は、
ぜひ`React.memo`と`useCallback`の導入を検討してみてください。
