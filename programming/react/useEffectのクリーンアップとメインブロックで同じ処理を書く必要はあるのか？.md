## useEffect のクリーンアップとメインブロックで同じ処理を書く必要はあるのか？

## はじめに

React の`useEffect`を使って定期的な処理（`setInterval`）を実装するとき、次のようなコードを見ることがあります：

```tsx
useEffect(() => {
  // メイン処理
  if (isRunning) {
    intervalRef.current = setInterval(() => {
      console.log("Running...");
    }, 1000);
  } else if (intervalRef.current) {
    window.clearInterval(intervalRef.current);
    intervalRef.current = null;
  }

  // クリーンアップ処理
  return () => {
    if (intervalRef.current) {
      window.clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };
}, [isRunning]);
```

一見すると `else if` ブロックと `return` のクリーンアップ関数が同じことをしているように見え、

> 「クリーンアップで止めるなら `else if` いらないのでは？」

と思う方も多いでしょう。

結論から言えば、**実はこの`else if`は不要です**。

@honey32 さん、いつもありがとうございます。

## useEffect の実行タイミングをおさらい

- **初回レンダリング時**：`useEffect`本体が実行される。
- **依存配列（例: `[isRunning]`）が変わるたび**：
  ① クリーンアップ関数が呼ばれ
  ② その後に`useEffect`本体が再実行される。
- **コンポーネントのアンマウント時**：最後にクリーンアップ関数が呼ばれる。

つまり、`isRunning` が `true → false` に変わったときは、**前回の Effect のクリーンアップ関数が呼ばれ、そこで`clearInterval`される**のです。

**✅ `else if` を省いたシンプルで安全な実装**

```tsx
useEffect(() => {
  if (!isRunning) return; // ここで早期に抜ける（《「処理の塊」を意識するための補助線》）

  const interval = window.setInterval(() => {
    setCount((prev) => prev + 1);
  }, 1000);

  return () => {
    window.clearInterval(interval);
  };
}, [isRunning]);
```

> **《「処理の塊」を意識するための補助線》として活用することをオススメします！**

「処理の塊」とは、

- `isRunning === true` なら「タイマーを動かす処理」
- それ以外のときは「なにも処理しない」
- `isRunning === true` のときだけ `setInterval`
- `false` になったときは、**前回 Effect のクリーンアップで自動的に停止**

これで十分機能します。**`else if`ブロックは不要**です。

## すぐ止めたいから else if が要るのでは？

「ボタンを押して `isRunning = false` にした瞬間、**クリーンアップが走るまでの間にレースコンディションが起こるのでは？**」という懸念があるかもしれません。

しかし、React は依存配列が変化した際：

1. クリーンアップ関数を**先に必ず**実行
2. その後で Effect 本体を再実行（今回で言えば何もせず return）

この順序は保証されています。したがって、**「クリーンアップより先に何か起こってリークする」ような race condition は起こりません**。

また、`intervalRef`のような変数も、**クリーンアップ内で止めていれば残っていても実害はなく、次の起動で上書きされるだけ**です。

**🚨 例外**
`intervalRef.current` を他の処理でも参照している場合

以下のように、Effect 外で `intervalRef.current` を参照して条件分岐しているケースでは、明示的に `null` にした方が安全です：

```tsx
if (intervalRef.current) {
  // タイマー中かどうかの判定に使っている
}
```

このようなケースではクリーンアップ内で `intervalRef.current = null` を明示的に書くことは有意義です。

## 🧩 補足：`setInterval` vs `window.setInterval`

前述のタイマー処理に関連して、`setInterval` の使い方にも注意点があります。

```tsx
// ❌ Node.js の型になる可能性あり
const intervalRef = useRef<NodeJS.Timeout | null>(null);
intervalRef.current = setInterval(...);

// ✅ ブラウザで確実に number 型になる
const intervalRef = useRef<number | null>(null);
intervalRef.current = window.setInterval(...);
```

TypeScript 環境では、`setInterval` は実行環境により戻り値の型が異なり、
**Node.js 環境では `NodeJS.Timeout`、ブラウザでは `number`** になります。

React + ブラウザ前提のコードでは、**意図的に `window.setInterval` を使うことで型を明確に `number` に固定**できます。

💡 `clearInterval` でも同様に `window.clearInterval` を使うのがおすすめです。

## おわりに

eact の`useEffect`は、「セットアップとクリーンアップが 1 つの処理の塊」である、という考え方が大切です。

処理を明示的に分けすぎてしまうと、React の自然なフローを逆行するような冗長なコードになります。

「**false になったら止まる**」「**クリーンアップは必ず呼ばれる**」というルールを信頼して、よりシンプルで読みやすいコードを書いていきましょう！
