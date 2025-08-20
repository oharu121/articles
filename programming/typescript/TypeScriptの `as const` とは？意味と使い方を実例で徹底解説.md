## はじめに

TypeScript のコードを読んでいると、次のような `as const` という記述を見かけることがあります。

```ts
const fruits = ["apple", "banana", "orange"] as const;
```

初めて見る人は

> 「`const` なのに、さらに `as const` って何？」

と疑問に思うかもしれません。
本記事では、**`as const` が何をしているのか、どう便利なのか** を例を交えて解説します。

## `as const` の意味

`as const` は、**オブジェクトや配列、文字列リテラルの型を「リテラル型」として固定する」型アサーション**です。

通常、変数は「もっと広い型」に推論されます。
`as const` を付けることで、値の型を「文字通りの値そのもの」に固定できます。

### 1\. `as const` なしの場合

```ts
const fruits = ["apple", "banana", "orange"];
// 型は string[] と推論される

fruits[0] = "grape"; // OK（型的には許可される）
```

`string[]` という広い型なので、 `"grape"` のような別の文字列も代入できます。

```ts
const user = { name: "Alice", role: "admin" };
// 型は { name: string; role: string }

user.name = "Bob"; 
// OK（型的には許可される）
```

### 2\. `as const` ありの場合

```ts
const fruits = ["apple", "banana", "orange"] as const;
// 型は readonly ["apple", "banana", "orange"] になる

fruits[0] = "grape"; 
// ❌ エラー: 読み取り専用のため変更できない
```

```ts
const user = { name: "Alice", role: "admin" } as const;
// 型は { readonly name: "Alice"; readonly role: "admin" }

user.name = "Bob"; 
// ❌ エラー
```

`as const` を付けると型が**リテラル型**になり、さらに配列やオブジェクトも**読み取り専用 (readonly)** になります。

## 実用例

### 1\. ユニオン型の生成

`as const` は配列からユニオン型を作るときによく使われます。

```ts
const roles = ["admin", "user", "guest"] as const;
type Role = typeof roles[number];
// "admin" | "user" | "guest"
```

これにより、特定の文字列だけを許可する型を簡単に作れます。

### 2\. switch文での型安全

```ts
const statuses = ["success", "error", "loading"] as const;
type Status = typeof statuses[number];

function setStatus(status: Status) {
  switch (status) {
    case "success":
      console.log("成功");
      break;
    case "error":
      console.log("失敗");
      break;
    case "loading":
      console.log("読み込み中");
      break;
    // ここで default が不要になる（網羅性チェックが効く）
  }
}

setStatus("success"); // ✅
setStatus("pending"); // ❌ エラー
```

**‼️ 注意点**

>* `as const` は型を固定するため、後から値を変更する予定がある変数には向かない
>* `readonly` になるので、ミュータブルな配列・オブジェクト操作はできなくなる

## まとめ

>* `as const` は値の型を「リテラル型」に固定し、配列やオブジェクトを `readonly` にする。
>* 主な用途は**定数配列からユニオン型を作る**、**型安全なコードを書く**ため。
>* 「後から変更しない値」に対して型安全性を高めるために使う。

**💡 一言でいうと**

`as const` は「値をそのままの形で型にする魔法のキーワード」です。
これを知っていると、TypeScript でより安全なコードが書けます。
