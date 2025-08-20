## `useRef` ：再レンダリングしない “もうひとつの状態”

## はじめに

React を学び始めた人が最初に出会うフックといえば `useState`。でも、ちょっと慣れてくると見かけるのが `useRef`。本記事では、そんな `useRef` の正体と使い方を、`useState` との違いを軸に丁寧に解説します。

## `useRef` とは

- `useRef` は「再レンダリングされない状態」のようなもの。
- 値を保持しても **再描画を引き起こさない**。
- `useRef().current` に好きな値を入れて、**後から自由に書き換え可能**。
- DOM 要素の参照以外にも、**値の保持や外部との同期**など様々な用途に使える。

**⚠️ `useRef` を使うときの注意点**

- `.current` の変更では **再レンダリングされない**ため、UI に反映したいなら `useState` と併用する。
- `useRef` は React の **描画ロジックに関与しない**ので、「純粋な状態管理」には向かない。
- 初期値は `null` にするのが一般的（DOM ノードなど未定義の状態が多いため）。

**`useState` vs `useRef`：違いは何？**

| 特性                         | `useState`    | `useRef`                     |
| ---------------------------- | ------------- | ---------------------------- |
| 値の更新で再レンダリング     | ✅ あり       | ❌ なし                      |
| 値の保持                     | ✅ 可能       | ✅ 可能                      |
| ミュータブル（直接書き換え） | ❌ 不可       | ✅ 可能（`.current` に代入） |
| 主な用途                     | UI の状態管理 | 値の参照・保持、DOM 操作など |

## 具体的なユースケース

#### 1. タイマー ID の保持

```tsx
const timerRef = useRef<number | null>(null);

function startTimer() {
  timerRef.current = window.setTimeout(() => {
    console.log("タイマー終了");
  }, 1000);
}

function cancelTimer() {
  if (timerRef.current !== null) {
    clearTimeout(timerRef.current);
  }
}
```

#### 2. DOM ノードの参照

```tsx
const inputRef = useRef<HTMLInputElement>(null);

function focusInput() {
  inputRef.current?.focus();
}

return <input ref={inputRef} />;
```

#### 3. 外部ライブラリとの同期

```tsx
const chartRef = useRef<Chart | null>(null);

useEffect(() => {
  chartRef.current = new Chart(canvasRef.current!, {
    /* options */
  });

  return () => chartRef.current?.destroy();
}, []);
```

#### 4. 前回の値を保持する

```tsx
function usePrevious<T>(value: T): T | undefined {
  const prevRef = useRef<T>();

  useEffect(() => {
    prevRef.current = value;
  }, [value]);

  return prevRef.current;
}
```

#### 5. 定期的に最新状態を取得する

```tsx
// 最新のfetch関数をRefに格納
React.useEffect(() => {
  fetchNotificationsRef.current = async () => {
    const res = await fetch(`/api/notifications?user=${username}`);
    const data = await res.json();
    setCount(data.count);
  };
}, [username]); // usernameが変わるたびに更新

// 5秒ごとにユーザの通知情報を取得する
React.useEffect(() => {
  const id = setInterval(() => {
    fetchNotificationsRef.current?.();
  }, 5000);

  return () => clearInterval(id);
}, []);
```

## まとめ

`useRef` は、「再レンダリングしない状態」として非常に便利なフックです。`useState` と違って描画をトリガーしないため、**パフォーマンスを保ちつつ値を保持したい**場面で活躍します。

- 値を保存したいけど UI に関係ない
- DOM ノードを直接触りたい
- 一時的な外部データと同期したい

そんなときには、`useRef` の出番です。
