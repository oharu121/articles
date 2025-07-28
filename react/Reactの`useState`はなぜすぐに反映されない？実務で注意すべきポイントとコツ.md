# はじめに

Reactで以下のようなコードを書くと、初心者から上級者まで一度は「なんでこうなるの？」と混乱します。

```tsx
const [status, setStatus] = React.useState("clean")

const handleClick = () => {
  setStatus("dirty")
  alert(status) // えっ、まだ"clean"なの？
}
```

ボタンをクリックすると`status`を`"dirty"`に設定したはずなのに、`alert`で表示されるのは古い値の`"clean"`──これにはちゃんと理由があります。この記事ではその理由と、**実務での注意点や対応方法**を紹介します。

# Reactの状態更新は「非同期」

Reactでは`useState`による状態変更は**即座に反映されるわけではありません**。代わりに「再描画のスケジュール」がされるだけです。これは以下のような流れです：

1. `setStatus("dirty")`が呼ばれる
2. 状態更新がReactにキューイングされる
3. `handleClick`の実行が終わったあとに再レンダリングが発生
4. そのときようやく`status === "dirty"`となる

つまり、`setStatus`直後に`status`を参照しても**まだ古い値のまま**というわけです。

# 実務での注意点とコツ

### ✅ 1. 状態変更後の処理は`useEffect`で

状態が変わった「あと」に処理を行いたい場合、`useEffect`を使うのが正しいアプローチです。

```tsx
useEffect(() => {
  if (status === "dirty") {
    alert("状態が変更されました: " + status)
  }
}, [status])
```

### ✅ 2. `setX`の直後に`X`を使わない

ありがちなバグ：

```tsx
setIsLoading(true)
console.log(isLoading) // まだfalse
```

`isLoading`はまだ古い値のままです。こうした処理のタイミングミスは非同期通信（APIリクエスト）やUIトグルで特に起こりやすくなります。

### ✅ 3. 状態の変更が即時必要な場合はローカル変数や関数内で扱う

「すぐに使いたい値」は状態で持つのではなく、**ローカル変数**で完結することも検討しましょう。

```tsx
const handleClick = () => {
  const nextStatus = "dirty"
  alert(nextStatus) // これは確実に"dirty"
  setStatus(nextStatus)
}
```

### ✅ 4. フォームや一時的なUIは「コントロールされすぎない」ように設計する

複雑なフォームやモーダル制御で状態の変更タイミングが混乱しやすい場合、以下のような工夫をしましょう：

* 表示・非表示を制御するstateをbooleanで持つ（`isOpen`など）
* フォームの一時入力状態は`useRef`や`controlled input`で管理
* 状態変化によって動く処理は**必ず副作用として`useEffect`で定義**

**🧪 おまけ：再現用ミニコンポーネント**

```tsx
export default function VibeCheck() {
  const [status, setStatus] = React.useState("clean")

  const handleClick = () => {
    setStatus("dirty")
    alert(status) // 1回目は "clean" が出る
  }

  return (
    <button onClick={handleClick}>
      {status}
    </button>
  )
}
```

**💡 併せて読みたい**

https://qiita.com/oharu121/items/a910d2dbd281476d718f

# おわりに

Reactの状態管理は慣れれば非常に強力ですが、最初はその**非同期性**に戸惑うことも多いです。この記事が、「なぜ古い値が表示されるのか？」という疑問のヒントになれば幸いです。
