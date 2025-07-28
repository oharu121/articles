# はじめに

Reactで`addEventListener`を使うとき、こんなふうに書いたことはありませんか？

```tsx
const handleScroll = () => {
  console.log("scroll")
}

useEffect(() => {
  window.addEventListener("scroll", handleScroll)
  return () => {
    window.removeEventListener("scroll", handleScroll)
  }
}, [])
```

一見これで十分に思えますが、実はこの書き方、Reactでは**危険な罠**を含んでいます。

本記事では、**Reactでイベントリスナーを安全に扱うためのコツ**を、**2つの代表的な罠**とその**実践的な解決法**を通じて整理します。

# 1. イベントリスナーの関数参照問題

**❌ 何が問題？**

JavaScriptでは、`removeEventListener` は `addEventListener` に渡したのと **「同じ関数参照」** でなければリスナーを削除できません。

しかし、Reactでは関数コンポーネントが再レンダリングされるたびに `handleScroll` のような関数は**毎回新しく生成される**ため、**参照が一致しない**可能性があります。

```tsx
const handleScroll = () => {
  console.log("scroll")
}

useEffect(() => {
  window.addEventListener("scroll", handleScroll)
  return () => {
    window.removeEventListener("scroll", handleScroll) // ❌ 別参照になっているかも
  }
}, [])
```

**✅ 解決法1：`useCallback`で関数をメモ化**

```tsx
const handleScroll = useCallback(() => {
  console.log("scroll")
}, [])

useEffect(() => {
  window.addEventListener("scroll", handleScroll)
  return () => {
    window.removeEventListener("scroll", handleScroll)
  }
}, [handleScroll])
```

`useCallback` を使うことで、**同じ関数参照が保たれ、登録と解除が一致**します。

**✅ 解決法2：`useEffect`の中で定義する**

```tsx
useEffect(() => {
  const handleScroll = () => {
    console.log("scroll")
  }

  window.addEventListener("scroll", handleScroll)
  return () => {
    window.removeEventListener("scroll", handleScroll)
  }
}, [])
```

同じスコープで定義されることで、**`add`と`remove`の関数参照が常に一致**する安全な書き方です。

# 2. stateの最新値が保証されない

たとえば、タブの切り替え検出イベントでカウントを増やす場合：

```tsx
useEffect(() => {
  const handleChange = () => {
    if (document.visibilityState !== "visible") {
      setCount(count + 1) // ❌ 最新のcountかはわからない
    }
  }

  document.addEventListener("visibilitychange", handleChange)
  return () => {
    document.removeEventListener("visibilitychange", handleChange)
  }
}, [count]) // ❗ 依存にcountを入れるとEffectが再実行される
```

**❌ 問題1：クロージャで古い `count` を参照してしまう**

`count` はクロージャに閉じ込められており、イベント発火時にはすでに**古い値**になっている可能性があります。
その結果、**連続でイベントが発生しても1回しかカウントされない**などのバグが起こり得ます。

**❌ 問題2：依存配列に `count` を入れると、Effectが毎回再実行される**

`useEffect`の依存配列に `count` を含めると、`count` が更新されるたびに

* イベントリスナーが毎回登録・解除され
* 結果として**不要な再登録が繰り返される**

という**非効率な処理**になります。

**✅ 解決法：`useCallback`または`useEffect`内でhandlerを定義**

依存にstateを入れたくない場合は、handler内でstateを使わない書き方に変えるのがコツです。

```tsx
useEffect(() => {
  const handleChange = () => {
    if (document.visibilityState !== "visible") {
      setCount((prev) => prev + 1)
    }
  }

  document.addEventListener("visibilitychange", handleChange)
  return () => {
    document.removeEventListener("visibilitychange", handleChange)
  }
}, []) // ✅ countを依存に入れなくてOK
```

# まとめ

Reactでイベントリスナーを扱うときは、次の2つの罠に特に注意してください。

**🕳 罠1：リスナー関数の参照が毎回変わる**

* 🔧 解決法1：`useCallback`でメモ化
* 🔧 解決法2：`useEffect`内で関数を定義

**🕳 罠2：stateの最新値が取れない**

* 🔧 解決法：handler内でstateを直接使わない構成にする
