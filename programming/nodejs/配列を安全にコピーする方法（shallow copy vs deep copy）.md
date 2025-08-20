## 配列を安全にコピーする方法（shallow copy vs deep copy）

## はじめに

**こんな経験ありませんか？**

表計算アプリ、グリッドレイアウトなどを JavaScript や TypeScript で扱っているとき、「次の状態を作るために配列をコピーしたのに、**なぜか元の配列まで変わってしまう**」

そんな不思議なバグに遭遇したことはありませんか？

```ts
const current = [
  [0, 1],
  [1, 0],
];

const next = [...current]; // コピーしたつもりが…
next[0][0] = 999;

console.log(current[0][0]); // 👉 999!? なぜ？
```

実はこれ、「**shallow copy（浅いコピー）**」が原因なんです。

## Shallow copy（浅いコピー）とは？

shallow copy とは、**配列やオブジェクトの「最上位の層」だけをコピーすること**です。
それに含まれる**ネストされた配列やオブジェクトはコピーされず、参照（ポインタ）だけがコピーされます**。

```js
const original = [
  [0, 1],
  [1, 0],
];
const shallowCopy = [...original];

// shallowCopy[0] は original[0] と同じ参照を持っている
```

`shallowCopy[0][0] = 999` と書くと、**original\[0]\[0] にも影響が出る**というわけです。

## Deep copy（深いコピー）とは？

deep copy は、**配列やオブジェクトの中身まで含めてすべて新しく複製すること**です。

2 次元配列の場合、**各行（内側の配列）も新しいインスタンスにする必要があります**。

```ts
const current = [
  [0, 1],
  [1, 0],
];

// deep copy
const next = current.map((row) => [...row]);
next[0][0] = 999;

console.log(current[0][0]); // 👉 0（元の配列は変わらない）
```

## 次元ごとの正しいコピー方法

#### 1 次元配列（array）

```ts
const arr = [1, 2, 3];
const copy = [...arr]; // OK
```

#### 2 次元配列（grid）

```ts
const grid = [
  [1, 2],
  [3, 4],
];
const copy = grid.map((row) => [...row]); // ✅
```

#### 3 次元配列（cube）

```ts
const cube = [
  [
    [1, 2],
    [3, 4],
  ],
  [
    [5, 6],
    [7, 8],
  ],
];
const copy = cube.map((matrix) => matrix.map((row) => [...row])); // ✅
```

📌 一般ルール（再掲）

> 多次元配列のコピーは、**次元の数だけ map（またはループ）をネストしてコピーする**のが基本です。

## 🛠 補足：再帰やライブラリを使う場合

もっと深い構造（ネストされたオブジェクトや配列の混在など）の場合は、以下のような方法が便利です。

**標準ブラウザ API の `structuredClone`**

- ブラウザや Node.js が提供する**ネイティブな deep copy API**。
- パフォーマンスが非常に良く、**軽量で高速**。
- ただし、**コピーできない型**がある（例：関数、プロトタイプ、DOM ノード、Map のキーがオブジェクトなど）。

```ts
function deepCopy<T>(value: T): T {
  return structuredClone(value);
}
```

**ライブラリ依存の `lodash.cloneDeep`**

- JavaScript のあらゆる型に対応しており、**堅牢性が高い**。
- React や Vue など、さまざまな環境で実績あり。
- ライブラリ依存なので、バンドルサイズが気になる場合は注意。

```ts
import cloneDeep from "lodash.clonedeep";

const copy = cloneDeep({ a: 1, b: [1, 2, 3] });
```

## おわりに

- `next = [...current]` は shallow copy。ネストされた配列は**コピーされず、参照のまま**。
- 2 次元以上の配列では `.map(row => [...row])` などで**深くコピー**する。
- 「コピーしたのに元のデータが壊れる」現象の多くは shallow copy が原因。

バグの原因が「コピー方法のミス」だった…という経験、ぜひあなたは回避してください！
