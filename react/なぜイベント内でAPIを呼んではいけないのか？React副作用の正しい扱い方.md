# なぜイベント内でAPIを呼んではいけないのか？React副作用の正しい扱い方

# はじめに

Reactでアプリを開発していると、つい「ボタンを押したらAPIを叩く」といったコードを書きたくなります。しかし、それは拡張性・安定性の観点から**危険なパターン**になることがあります。

本記事では、Reactで副作用（特にAPI通信）をどう扱うべきか、「なぜイベント内でAPIを呼ぶのが問題なのか」をポケモンAPIを例に丁寧に解説します。

# ❌一見正しく見える実装

```tsx
<button onClick={fetchNextPokemon}>次のポケモン</button>

async function fetchNextPokemon() {
  const res = await fetch(`https://pokeapi.co/api/v2/pokemon/2`);
  const data = await res.json();
  setPokemon(data);
}
```

これは動きますが、次のような課題が潜んでいます：

* 初回表示でポケモンを取得したい場合、別途useEffectが必要
* 自動でポケモンを切り替えるにはsetIntervalが必要
* キーボードでの操作や別UIでの制御がしにくい

つまり、**操作に対してfetchする形だと、仕様が増えるたびにロジックが複雑になる**のです。

# ✅状態をトリガーとしてfetchする

正しいアプローチは、「状態（例：`id`）が変わったらfetchする」形にすることです。

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

* 初期レンダリングで自動fetch
* `id`の更新で再fetch（UIのどこからでも）
* setIntervalでもキーボードでも制御しやすい

と、**状態を中心にした同期処理**が可能になります。

# ポケモンカルーセルの例

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

このように、**fetch自体はuseEffectに任せて、イベントでは状態変更だけを行う**設計にすることで、アプリの柔軟性・再利用性が格段に上がります。

# まとめ

| アプローチ                  | 問題点                |
| ---------------------- | ------------------ |
| イベント内でAPIを叩く           | 初期表示・自動更新・他UI連携に弱い |
| 状態変化でuseEffectからAPIを叩く | 柔軟で拡張性が高い          |

副作用のタイミングは**ユーザーイベントではなく「状態の変化」によって制御**しましょう。これがReactの設計哲学に沿った、安全で拡張しやすい方法です。
