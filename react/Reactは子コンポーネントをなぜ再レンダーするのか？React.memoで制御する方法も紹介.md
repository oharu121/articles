# はじめに

Reactで開発していて、こんな経験はありませんか？

> 「親コンポーネントのstateを更新しただけなのに、関係なさそうな子コンポーネントまで再レンダーされてる…？」

今回は、**propsを受け取っていない子コンポーネントが再レンダーされる理由**と、**それを防ぐ方法（`React.memo`）**を、できるだけシンプルなコード例と**ログ出力の挙動**を使って紹介します。

# 実験：最小構成で確認してみよう

まずは以下の2つのコンポーネントを見てください。

```jsx:App.jsx
import React, { useState } from "react";
import Message from "./Message";

export default function App() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount((prev) => prev + 1);
  };

  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={handleClick}>+1</button>
      <Message />
    </div>
  );
}
```

```jsx:Message.js
import React from "react";

export default function Message() {
  console.log("👋 Message rendered");
  return <p>Hello!</p>;
}
```

* `App` はカウント用の state（`count`）を持っています。
* `Message` は props を一切受け取っていません。

**🖥 ログ出力の挙動**

ブラウザの開発者ツールのコンソールを開いて、ボタンを3回クリックしてみましょう。

```log
👋 Message rendered
👋 Message rendered
👋 Message rendered
👋 Message rendered
```

※ 初回マウント + クリック3回分 = 計4回のログが出力されます。

# なぜ props がないのに再レンダーされるの？

Reactの基本ルールはこうです：

> **stateが変わると、そのコンポーネントとすべての子が再レンダーされる**

つまり、たとえ `Message` に props がなく、何も変化していなくても、`App` の再レンダーに巻き込まれてしまうのです。

# `React.memo` で再レンダーを防ぐ

このような「見た目もpropsも変わらないのに毎回レンダーされる」のは、場合によってはパフォーマンスの無駄になります。

そんなときに使えるのが `React.memo` です。

```jsx:Message.jsx
import React from "react";

function Message() {
  console.log("👋 Message rendered");
  return <p>Hello!</p>;
}

export default React.memo(Message);
```

**✅ ログ出力の変化を確認してみよう**

`React.memo` 適用後に同じようにボタンを3回クリックすると、コンソールは次のようになります。

```log
👋 Message rendered
```

たった1回だけです。これは、**初回マウント時のログのみ**で、以降は再レンダーされていないことを意味します。

# `React.memo` は何にでも使ってよい？

いいえ。次のようなケースで使うのが効果的です。

* **重たい処理を行っているコンポーネント**
* **再レンダーのコストが大きいUI**
* **レンダーの頻度を下げたい場合**

軽いUIであれば無理に使う必要はありません。**パフォーマンスのボトルネックになってから導入する方が自然**です。

#  まとめ

* 親コンポーネントのstateが変わると、**propsが変わらなくても子も再レンダーされる**
* 小さなアプリでは問題になりにくいが、大きなアプリでは再レンダーのコストが気になる場合もある
* `React.memo` を使うと、**propsが変わらない限り再レンダーを防ぐことができる**
