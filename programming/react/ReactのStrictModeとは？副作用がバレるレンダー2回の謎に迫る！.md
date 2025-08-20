## React の StrictMode とは？副作用がバレるレンダー 2 回の謎に迫る！

## はじめに

React を使っていると、開発中に思わぬ **「2 回レンダーされる」** 現象に出会ったことはありませんか？それ、もしかしたら `StrictMode` が原因かもしれません。

本記事では、React の`StrictMode`が何をしているのか、なぜレンダーが 2 回発生するのか、パフォーマンスへの影響は？といった疑問に答えていきます。

## StrictMode ってなに？

`StrictMode` は、React が提供する**開発モード専用の検証ツール**です。

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";

const root = createRoot(document.getElementById("root")!);

root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

**✅ やってくれること**

- 副作用（`useEffect` など）の**意図しない挙動**を検出
- コンポーネントの **純粋性（Pure Component）** のチェック
- 非推奨 API の使用を警告

## レンダーが 2 回起きるのはなぜ？

`StrictMode`を使うと、**関数コンポーネントや`useEffect`などの副作用が一度「アンマウント → 再マウント」されます**。これは意図的な仕様です。

```tsx
function Wave() {
  useEffect(() => {
    console.log("👋 Wave mounted");

    return () => {
      console.log("👋 Wave unmounted");
    };
  }, []);

  return <div>👋</div>;
}
```

StrictMode でこのコンポーネントを使うと、**以下のログが出ます**：

```
👋 Wave mounted
👋 Wave unmounted
👋 Wave mounted
```

これは、「副作用が再実行されても安全か？」を検証するための**開発用チェック**です。

**⚠️ パフォーマンスへの影響は？**

心配しなくて大丈夫です。`StrictMode`の影響は**開発モードだけ**です。**本番ビルドでは無効化され、自動的に除外されます。**

## どう使えばいいの？

基本的に、開発中は`StrictMode`を**有効にしておくのが推奨**されます。

**✔️ こんなときに便利**

- useEffect や useRef の扱いに自信がないとき
- チーム開発で不具合の原因を早期発見したいとき
- 複雑な状態管理を使っていて、レンダーの副作用が気になるとき

| ポイント                                     | 説明                                  |
| -------------------------------------------- | ------------------------------------- |
| ✅ `StrictMode`は開発専用                    | 本番には影響なし                      |
| 🔁 副作用を検証するために 2 回レンダーされる | 安全性・純粋性を確認できる            |
| 🛠️ 有効にしても問題ないアプリが理想          | React の思想「UI は状態の関数」に合致 |

## おわりに

React の`StrictMode`は、開発者が**より堅牢なコンポーネント設計**をするための道しるべです。最初はレンダーが 2 回起きるのに戸惑うかもしれませんが、「UI は状態の関数」という React の哲学を実感できるよい機会でもあります。
