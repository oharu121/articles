# **はじめに**

Reactでコンポーネントを扱う際、`onClick` や `onChange` といったイベントハンドラに必ずと言っていいほど登場する引数 `(e)`。

「とりあえず書いてはいるけど、この `e` って一体何者？」
「`e.target.value` は何となくわかるけど、他に何ができるの？」

そんな疑問を抱えている方のために、この記事ではReactの\*\*イベントオブジェクト（通称`e`）\*\*の正体と、その実用的な使い方を基礎から徹底的に解説します。

# `e`の正体はReact独自の「合成イベント」

イベントハンドラに渡される `e` の正体は、**SyntheticEvent（合成イベント）** と呼ばれるオブジェクトです。

これは、Chrome, Safari, Firefoxなど、ブラウザごとに挙動が微妙に異なるネイティブのDOMイベントをReactがラップし、**どのブラウザでも同じように扱える**ようにしたものです。このおかげで、開発者はブラウザ間の差異を気にすることなく、一貫した方法でイベントを処理できます。

```jsx
const handleClick = (e) => {
  // この 'e' が SyntheticEvent オブジェクト
  console.log(e); 
};

return <button onClick={handleClick}>Click Me</button>;
```

# 最もよく使うプロパティとメソッド TOP3

SyntheticEvent には多くの情報が含まれていますが、日常的な開発で使うものは限られています。まずは最重要の3つをマスターしましょう。

### 1. `e.target`：イベントが発生した要素を取得

`e.target` は、**イベントが実際に発生したDOM要素**を指します。フォームの入力値を扱う際に最もよく使われます。

>* `e.target.value`: `<input>` や `<textarea>`, `<select>` の現在の値を取得します。
>* `e.target.name`: 要素の`name`属性。複数のinputを一つの関数で管理する際に便利です。
>* `e.target.checked`: チェックボックスやラジオボタンが選択されているかを `true`/`false` で取得します。

<!-- end list -->

```jsx:UserForm.jsx
function UserForm() {
  const [name, setName] = useState('');

  const handleChange = (e) => {
    // e.target は <input> 要素そのものを指す
    // e.target.value はその入力値
    setName(e.target.value);
  };

  return (
    <input 
      type="text" 
      value={name} 
      onChange={handleChange} 
      placeholder="名前を入力" 
    />
  );
}
```

### **2. `e.preventDefault()`：要素のデフォルト動作をキャンセル**

このメソッドを呼び出すと、ブラウザが標準で実行する動作をキャンセルできます。

代表的な例は、`<form>` タグ内での送信ボタンクリックです。通常、フォーム送信ボタンを押すとページがリロードされますが、`e.preventDefault()` を使うことで、**ページ遷移を防ぎ、代わりにAPIリクエストなどの非同期処理を実行する**ことができます。

```jsx:SearchForm.jsx
function SearchForm() {
  const handleSubmit = (e) => {
    // formのデフォルトの送信（ページリロード）をキャンセル
    e.preventDefault(); 
    
    alert('検索処理をJavaScriptで実行します！');
    // ここで入力値を使ったAPIリクエストなどを行う
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="search" />
      <button type="submit">検索</button>
    </form>
  );
}
```

### **3. `e.stopPropagation()`：イベントの伝播を停止**

HTML要素が入れ子になっている場合、内側の要素で発生したイベントは、外側の親要素へと伝播（バブリング）していきます。`e.stopPropagation()` はこの伝播を途中で止めたいときに使います。

```jsx:ParentComponent.tsx
function ParentComponent() {
  const handleParentClick = () => {
    alert('親要素のイベントが発火しました');
  };
  
  const handleChildClick = (e) => {
    // この一行がないと、子をクリックした後、親のアラートも表示される
    e.stopPropagation(); 
    alert('子要素のイベントだけを発火させます');
  };

  return (
    <div onClick={handleParentClick} style={{ padding: '20px', backgroundColor: 'lightgray' }}>
      <button onClick={handleChildClick}>Click Me</button>
    </div>
  );
}
```

# その他の便利なプロパティ

TOP3以外にも、特定の状況で役立つプロパティがあります。

### 1. KeyboardEvent (`onKeyDown`, `onKeyUp`)

>* `e.key`: 押されたキーの識別子（例: `"Enter"`, `"a"`, `"ArrowUp"`）。特定のキー操作（Enterで送信など）を実装する際に使います。
>* `e.code`: 物理的なキーコード（例: `"KeyA"`, `"Enter"`）。言語設定に影響されません。

### 2. MouseEvent (`onClick`, `onMouseMove`)

>* `e.clientX`, `e.clientY`: ウィンドウの表示領域の左上を基準としたマウスの座標を取得します。

**⚠️【重要】React 17以降の変更点：`e.persist()`はもう不要**

以前のバージョンのReactでは、パフォーマンス上の理由からイベントオブジェクトが再利用されていました（イベントプーリング）。そのため、非同期処理（`setTimeout`など）の中でイベントのプロパティにアクセスするには、`e.persist()` を呼び出してイベントオブジェクトを保持する必要がありました。

しかし、**React 17以降、このイベントプーリングの仕組みは廃止されました。**

したがって、**現代のReact開発において `e.persist()` を呼び出す必要は基本的にありません。** 非同期処理の中でも安全に `e.target.value` などにアクセスできます。

```jsx
// React 17以降では、e.persist()は不要！
const handleChangeAsync = (e) => {
  const value = e.target.value; // 先に値を変数に保存しておくのがシンプルで良い

  setTimeout(() => {
    // 非同期処理内でも問題なく値にアクセスできる
    console.log(value); 
  }, 1000);
};
```

# 🚀 実装例：「連打検出ゲーム」

`e.timeStamp` プロパティを使って、簡単なゲームを作ってみましょう。これはイベントが発生した時刻（ページロードからのミリ秒）を返します。

```tsx:RapidFireGame.tsx
import React, { useState } from "react";

export default function RapidFireGame() {
  const [timestamps, setTimestamps] = useState<number[]>([]);
  const TIME_LIMIT = 1000; // 1秒
  const CLICK_GOAL = 5;     // 5回クリック

  const handleClick = (e: React.MouseEvent) => {
    const now = e.timeStamp;
    
    // 現在のクリック時刻を追加し、1秒以上前の古い記録をフィルタリング
    const updatedTimestamps = [...timestamps, now].filter(
      (ts) => now - ts < TIME_LIMIT
    );

    if (updatedTimestamps.length >= CLICK_GOAL) {
      alert(`💥 ${CLICK_GOAL}回成功！あなたの勝ちです！`);
      setTimestamps([]); // リセット
    } else {
      setTimestamps(updatedTimestamps);
    }
  };

  return (
    <div>
      <h2>🔥 連打ゲーム！</h2>
      <p>1秒以内に{CLICK_GOAL}回クリックして勝利しよう</p>
      <button onClick={handleClick}>Click Me!</button>
      <p>現在の有効打数: {timestamps.length}</p>
    </div>
  );
}
```

# **まとめ**

>* `e` はブラウザ差異を吸収したReactの**SyntheticEvent**オブジェクト。
>* **`e.target`** を使えば、イベントが発生した要素の値や属性を取得できる（最重要）。
>* **`e.preventDefault()`** はフォーム送信などのデフォルト動作をキャンセルする。
>* **`e.stopPropagation()`** は親要素へのイベント伝播を止める。
>* **React 17以降では `e.persist()` は不要**になった。

イベントオブジェクト `e` を使いこなすことは、インタラクティブなUIを構築するための第一歩です。ぜひこれらの知識を活用して、よりリッチなReactアプリケーションを開発してください。