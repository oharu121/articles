## 従来の DOM ⚔️ 仮想 DOM 〜そもそも React の Virtual DOM ってなに？〜

## はじめに

React に入門すると、必ず出てくるのが「Virtual DOM（仮想 DOM）」という言葉。

でも…

- 「仮想ってなに？」
- 「コピーってどういう意味？」
- 「どうしてそんな面倒な仕組みが必要なの？」

…と思ったことはありませんか？

この記事では、**従来の DOM 操作**と**React の Virtual DOM**の違いを丁寧に比較しながら、**Virtual DOM の正体とメリット**をわかりやすく解説します。

## 1. 従来の DOM 操作とは？

✅ 特徴

- `document.getElementById` などの DOM API を使って、HTML 要素を**直接操作**
- 状態と UI の整合性を**自分で管理する必要がある**

💡 Vanilla JS でカウンター

```html
<div id="count">0</div>
<button onclick="increment()">＋</button>

<script>
  let count = 0;
  function increment() {
    count++;
    document.getElementById("count").innerText = count;
  }
</script>
```

⚠️ 課題

- 命令的（imperative）で、更新処理が**散らかりやすい**
- 規模が大きくなると**保守が大変**
- UI 更新のたびに DOM を操作 → **パフォーマンスが悪化**

## 2. React の Virtual DOM とは？

✅ 一言でいうと：

**本物の DOM の軽量なコピーを JavaScript 内に持ち、差分だけを画面に反映する仕組み**です。

🧠 Virtual DOM の「コピー」ってなに？

React は、画面に表示される DOM と同じ構造を、JavaScript のオブジェクトで表現しています。

**JSX で書かれたコード**

```jsx
function Hello() {
  return <h1>Hello World</h1>;
}
```

このコード、実は**JSX という特殊な構文**で書かれており、**ビルド時に以下のような JavaScript に変換されます：**

```js
import { jsx as _jsx } from "react/jsx-runtime";

function Hello() {
  return _jsx("h1", {
    children: "Hello World",
  });
}
```

つまり：

> **ブラウザが受け取るのは、あくまで JavaScript だけ！**

React は JSX を JavaScript の関数呼び出しに変換し、それを使って仮想 DOM ツリーを構築しているのです。これが「仮想的な DOM（Virtual DOM）」＝**軽量なコピー**ということです。

## 3. どう使われるの？（更新の流れ）

💡 React 版のカウンター（宣言的）

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <>
      <div>{count}</div>
      <button onClick={() => setCount(count + 1)}>＋</button>
    </>
  );
}
```

- 状態（`count`）が変わると、新しい仮想 DOM が生成され、変更差分だけを実 DOM に反映。
- これにより**状態と UI の同期が自動で取れる**。

**React では、状態が変わると…**

1. 新しい Virtual DOM ツリーを生成
2. 前の Virtual DOM と比較（diff）
3. 変わった部分だけ実 DOM に反映（patch）

🎯 結果：

- DOM 全体を更新する必要なし
- 無駄な描画が減るので**高速**
- コードは**宣言的に書ける**

🧾 比較まとめ

| 項目           | Traditional DOM            | React（Virtual DOM）          |
| -------------- | -------------------------- | ----------------------------- |
| UI 構築方法    | 命令的（imperative）       | 宣言的（declarative）         |
| 更新方式       | DOM を直接操作             | 仮想 DOM を比較し差分のみ更新 |
| 保守性         | 大規模になると煩雑         | コンポーネント単位で明確      |
| パフォーマンス | 多くの更新で重くなりがち   | 差分のみ描画で高速            |
| 状態管理       | 手動で整合性を取る必要あり | `useState`などで一元管理      |

## おわりに

Virtual DOM は「React って難しそう…」と思わせる用語のひとつかもしれません。

でもその正体は：

> **JS の中にある軽量な DOM のコピーを比較して、必要なとこだけを効率よく更新する仕組み**

と知れば、実はとても合理的な考え方です。

- 状態と UI のずれを**自動で管理**
- 差分描画による**高速パフォーマンス**
- コンポーネント指向で**再利用しやすい**

つまり、

✅ コードは書きやすく
✅ 保守はしやすく
✅ パフォーマンスも良い

三拍子そろった仕組みなのです。

React を使って開発する中で、Virtual DOM の存在を意識してみると、理解がグッと深まります！
