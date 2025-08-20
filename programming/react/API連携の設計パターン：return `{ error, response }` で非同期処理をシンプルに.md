## API 連携の設計パターン：return `{ error, response }` で非同期処理をシンプルに

## はじめに

React で API を呼び出すとき、こんな風に書いていませんか？

```tsx
try {
  const res = await fetch(url);
  const data = await res.json();
  setData(data);
} catch (err) {
  setError(err);
}
```

一見シンプルですが、実際のアプリではこれを複数箇所で繰り返し、**try/catch が乱立**してしまうことも。

この記事では、非同期 API の呼び出し方をより簡潔に、再利用性も高く書けるようにする **「`{ error, response }` 形式で返す」設計パターン** をご紹介します。

## 問題：try/catch があちこちに増える

API を複数呼び出すアプリや、再リクエスト処理などがあると、try/catch 構文がどんどん増えて、可読性が下がってしまいます。

また、error ハンドリングが不統一になったり、呼び出し側のコードが煩雑になります。

## 解決策

> API 関数の中でエラーハンドリングし、構造化して返す

例えばポケモン API を呼び出す関数を以下のように定義します：

```ts
async function fetchPokemon(id: number): Promise<{
  error: Error | null;
  response: any | null;
}> {
  try {
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);

    if (!res.ok) throw new Error(`Error fetching pokemon #${id}`);

    const data = await res.json();
    return { error: null, response: data };
  } catch (err) {
    return { error: err as Error, response: null };
  }
}
```

_呼び出し側はシンプルに_

```tsx
const { error, response } = await fetchPokemon(1);

if (error) {
  setError(error);
} else {
  setPokemon(response);
}
```

## このパターンのメリット

1. **呼び出し側のロジックが平坦になる**

try/catch を書かなくてよく、状態更新や UI 処理に集中できます。

2. **error 処理の一貫性**

すべての API が `{ error, response }` 形式で返ってくる前提なので、共通処理がしやすくなります。

3. **テストや mock が楽になる**

関数の出力を模倣しやすく、`{ error, response }` をそのまま返せばいいのでユニットテストが簡潔になります。

## カスタムフックとの組み合わせ例

この設計は `useEffect` やカスタムフックとの相性も抜群です：

```tsx
useEffect(() => {
  let ignore = false;

  async function fetchData() {
    setLoading(true);
    setError(null);

    const { error, response } = await fetchPokemon(id);

    if (!ignore) {
      if (error) setError(error);
      else setPokemon(response);
      setLoading(false);
    }
  }

  fetchData();
  return () => {
    ignore = true;
  };
}, [id]);
```

## 他の一般的な設計との比較

| 方法                           | 特徴                                                                 |
| ------------------------------ | -------------------------------------------------------------------- |
| `try/catch` を呼び出し元で書く | 標準的・明示的・柔軟。シンプルな用途では適切。                       |
| `{ error, response }` を返す   | 呼び出し側が平坦で扱いやすい。小中規模に向く。                       |
| `Result<T, E>` 型（fp-ts 等）  | より厳格な型安全と関数型スタイル。導入コストが高め。                 |
| React Query / SWR を使う       | 非同期管理・キャッシュ・リトライなどを自動化。中大規模では有力候補。 |

**🛠 改善アイデア**

**1. ジェネリクスで型安全を強化**

API の返却型を指定できるようにすることで、より堅牢に：

```ts
type ApiResult<T> = {
  error: Error | null;
  response: T | null;
};
```

**2. HTTP ステータスの保持**

追加情報を持たせたい場合は `status` を含める：

```ts
return { error: null, response: data, status: res.status };
```

**3. AbortController 対応**

必要に応じてキャンセル対応も組み込めます：

```ts
async function fetchData(signal?: AbortSignal): Promise<ApiResult<T>> {
  const res = await fetch(url, { signal });
}
```

## まとめ

| 従来のやり方                                 | このパターン                             |
| -------------------------------------------- | ---------------------------------------- |
| try/catch を呼び出し側で毎回書く             | API 関数の中で処理を閉じて構造化して返す |
| catch 漏れ・error 処理のばらつきが起きやすい | 呼び出し側は `error != null` の確認だけ  |

非同期処理が絡む React アプリでは、**副作用の責務を関数に閉じ込める**設計がとても重要です。

**🧠 補足**

> { error, data } パターンは React Query と親和性が高い

React Query や SWR では data と error という形で非同期の状態を表現します。
したがって、この設計は、軽量な React Query 的 API 設計とも言えます。
