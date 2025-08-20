## 副作用って何？React での 3 つのルールで理解しよう！

## はじめに

React を使っていると、`useEffect` や `localStorage`、API 呼び出しなど、いわゆる **副作用（side effect）** にぶつかることが多くなります。

でも、副作用をちゃんと整理できていないと…

- 無駄な再レンダリングが増える
- 初期表示がバグる
- サーバーサイドレンダリング（SSR）でエラーになる

などのトラブルにつながります。

この記事では、React で副作用を扱うときに大事な**3 つのルール**を、**簡単な例とともに**紹介します。

## 1️⃣ 描画中に副作用を実行しない

**❌ ダメな例**

> localStorage から直接読み込む

```tsx
const [name, setName] = useState(localStorage.getItem("name") || "");
```

**🧨 なにが問題？**

- `localStorage.getItem()` は副作用（外部の状態を読む）
- それを **`useState` の初期値として直接使う**と、React がコンポーネントを**描画するたびに副作用が実行される**
- SSR 環境では `localStorage` がないため、**エラーになる**

**✅ 解決策**

> useEffect を使う

```tsx
const [name, setName] = useState("");

useEffect(() => {
  const stored = localStorage.getItem("name");
  if (stored) {
    setName(stored);
  }
}, []);
```

副作用は **useEffect の中で、描画のあとに実行**することで回避できます。

## 2️⃣ 副作用は発生源（イベント）の中に置く

**❌ ダメな例**

> 値の保存を useEffect でやってしまう

```tsx
useEffect(() => {
  localStorage.setItem("count", count);
}, [count]);
```

**🧨 なにが問題？**

- 状態が変わるたびに **副作用が再実行**される
- 毎回保存が走るので、**無駄が多い**
- 本来は「ユーザーが何かしたとき」に保存したいのに、**変更のたびに保存されてしまう**

**✅ 解決策**

> イベントの中で処理する

```tsx
const handleClick = () => {
  const newCount = count + 1;
  setCount(newCount);
  localStorage.setItem("count", String(newCount));
};
```

これなら：

- 状態を更新するタイミングと副作用の発生が一致
- 副作用が **自然な場所（イベント内）** に置かれていて、意図が明確

## 3️⃣ 外部との同期は useEffect に書く

**❌ ダメな例**

> API リクエストをレンダリング中に実行しようとする

```tsx
function Message() {
  const data = fetch("/api/message").then((res) => res.json());

  return <p>{/* ここにデータを表示したい */}</p>;
}
```

**🧨 なにが問題？**

- **`fetch()` が毎回レンダリング中に実行される**
- レンダリング中に副作用が走ってしまっている
- 実行タイミングも予測しづらく、状態と合わなくなる

**✅ 解決策**

> useEffect で処理する

```tsx
function Message() {
  const [text, setText] = useState("");

  useEffect(() => {
    fetch("/api/message")
      .then((res) => res.json())
      .then((data) => setText(data.text));
  }, []);

  return <p>{text}</p>;
}
```

これなら：

- 副作用（API 通信）は**描画後に 1 回だけ実行**
- `message` を状態にセットすれば、React が再描画してくれる

## おわりに

副作用の扱い方は、React の中でもつまずきやすいポイントですが、
**この 3 つのルール**を覚えておくだけで、かなり安全でわかりやすいコードが書けるようになります。

> 副作用をどこに書くか迷ったら、
> 「描画中？ useEffect？ イベント中？」
> と自問してみてください！
