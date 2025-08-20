## なぜイベント内で API を呼んではいけないのか？React 副作用の正しい扱い方

## はじめに

React でアプリを開発していると、つい「ボタンを押したら API を叩く」といったコードを書きたくなります。しかし、それは拡張性・安定性の観点から**危険なパターン**になることがあります。

本記事では、React で副作用（特に API 通信）をどう扱うべきか、「なぜイベント内で API を呼ぶのが問題なのか」をポケモン API を例に丁寧に解説します。

## ❌ 一見正しく見える実装

```tsx
<button onClick={fetchNextPokemon}>次のポケモン</button>;

async function fetchNextPokemon() {
  const res = await fetch(`https://pokeapi.co/api/v2/pokemon/2`);
  const data = await res.json();
  setPokemon(data);
}
```

これは動きますが、次のような課題が潜んでいます：

- 初回表示でポケモンを取得したい場合、別途 useEffect が必要
- 自動でポケモンを切り替えるには setInterval が必要
- キーボードでの操作や別 UI での制御がしにくい

つまり、**操作に対して fetch する形だと、仕様が増えるたびにロジックが複雑になる**のです。

## ✅ 状態をトリガーとして fetch する

正しいアプローチは、「状態（例：`id`）が変わったら fetch する」形にすることです。

```tsx
const [id, setId] = useState(1);
const [pokemon, setPokemon] = useState(null);

useEffect(() => {
  async function fetchPokemon() {
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
    const data = await res.json();
    setPokemon(data);
  }
  fetchPokemon();
}, [id]);
```

これにより：

- 初期レンダリングで自動 fetch
- `id`の更新で再 fetch（UI のどこからでも）
- setInterval でもキーボードでも制御しやすい

と、**状態を中心にした同期処理**が可能になります。

## ポケモンカルーセルの例

```tsx
function Carousel() {
  const [id, setId] = useState(1);
  const [pokemon, setPokemon] = useState(null);

  useEffect(() => {
    async function fetchData() {
      const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
      const data = await res.json();
      setPokemon(data);
    }
    fetchData();
  }, [id]);

  return (
    <>
      <button onClick={() => setId(id - 1)}>前</button>
      <button onClick={() => setId(id + 1)}>次</button>
      <div>{pokemon?.name}</div>
    </>
  );
}
```

このように、**fetch 自体は useEffect に任せて、イベントでは状態変更だけを行う**設計にすることで、アプリの柔軟性・再利用性が格段に上がります。

## まとめ

| アプローチ                           | 問題点                               |
| ------------------------------------ | ------------------------------------ |
| イベント内で API を叩く              | 初期表示・自動更新・他 UI 連携に弱い |
| 状態変化で useEffect から API を叩く | 柔軟で拡張性が高い                   |

副作用のタイミングは**ユーザーイベントではなく「状態の変化」によって制御**しましょう。これが React の設計哲学に沿った、安全で拡張しやすい方法です。
