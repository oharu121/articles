#  はじめに

Reactに入門すると、必ず出てくるのが「Virtual DOM（仮想DOM）」という言葉。

でも…

* 「仮想ってなに？」
* 「コピーってどういう意味？」
* 「どうしてそんな面倒な仕組みが必要なの？」

…と思ったことはありませんか？

この記事では、**従来のDOM操作**と**ReactのVirtual DOM**の違いを丁寧に比較しながら、**Virtual DOMの正体とメリット**をわかりやすく解説します。

# 1. 従来のDOM操作とは？

✅ 特徴

* `document.getElementById` などのDOM APIを使って、HTML要素を**直接操作**
* 状態とUIの整合性を**自分で管理する必要がある**

💡 Vanilla JSでカウンター

```html
<div id="count">0</div>
<button onclick="increment()">＋</button>

<script>
  let count = 0;
  function increment() {
    count++;
    document.getElementById('count').innerText = count;
  }
</script>
```

⚠️ 課題

* 命令的（imperative）で、更新処理が**散らかりやすい**
* 規模が大きくなると**保守が大変**
* UI更新のたびにDOMを操作 → **パフォーマンスが悪化**

# 2. ReactのVirtual DOMとは？

✅ 一言でいうと：

**本物のDOMの軽量なコピーをJavaScript内に持ち、差分だけを画面に反映する仕組み**です。

🧠 Virtual DOMの「コピー」ってなに？

Reactは、画面に表示されるDOMと同じ構造を、JavaScriptのオブジェクトで表現しています。

**JSXで書かれたコード**

```jsx
function Hello() {
  return <h1>Hello World</h1>
}
```

このコード、実は**JSXという特殊な構文**で書かれており、**ビルド時に以下のようなJavaScriptに変換されます：**

```js
import { jsx as _jsx } from "react/jsx-runtime"

function Hello() {
  return _jsx("h1", {
    children: "Hello World"
  });
}
```

つまり：

> **ブラウザが受け取るのは、あくまでJavaScriptだけ！**

ReactはJSXをJavaScriptの関数呼び出しに変換し、それを使って仮想DOMツリーを構築しているのです。これが「仮想的なDOM（Virtual DOM）」＝**軽量なコピー**ということです。

# 3. どう使われるの？（更新の流れ）
💡 React版のカウンター（宣言的）

```jsx
import { useState } from 'react';

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

* 状態（`count`）が変わると、新しい仮想DOMが生成され、変更差分だけを実DOMに反映。
* これにより**状態とUIの同期が自動で取れる**。

**Reactでは、状態が変わると…**

1. 新しいVirtual DOMツリーを生成
2. 前のVirtual DOMと比較（diff）
3. 変わった部分だけ実DOMに反映（patch）

🎯 結果：

* DOM全体を更新する必要なし
* 無駄な描画が減るので**高速**
* コードは**宣言的に書ける**

🧾 比較まとめ

| 項目      | Traditional DOM | React（Virtual DOM） |
| ------- | --------------- | ------------------ |
| UI構築方法  | 命令的（imperative） | 宣言的（declarative）   |
| 更新方式    | DOMを直接操作        | 仮想DOMを比較し差分のみ更新    |
| 保守性     | 大規模になると煩雑       | コンポーネント単位で明確       |
| パフォーマンス | 多くの更新で重くなりがち    | 差分のみ描画で高速          |
| 状態管理    | 手動で整合性を取る必要あり   | `useState`などで一元管理  |

# おわりに

Virtual DOMは「Reactって難しそう…」と思わせる用語のひとつかもしれません。

でもその正体は：

> **JSの中にある軽量なDOMのコピーを比較して、必要なとこだけを効率よく更新する仕組み**

と知れば、実はとても合理的な考え方です。

* 状態とUIのずれを**自動で管理**
* 差分描画による**高速パフォーマンス**
* コンポーネント指向で**再利用しやすい**

つまり、

✅ コードは書きやすく
✅ 保守はしやすく
✅ パフォーマンスも良い

三拍子そろった仕組みなのです。

Reactを使って開発する中で、Virtual DOMの存在を意識してみると、理解がグッと深まります！
