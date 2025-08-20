## Object と Array の使い方

## 1. Array から空の要素を除く

```javascript
const array = ["foo", "boo", "", "bar"];
const newArray = array.filter(Boolean); //["foo", "boo", "bar"]

const paragraph = `これは一行目です
これは二行目です

これは三行目です
これは四行目です
`;

const sentences = paragraph.split("\n").filter(Boolean);
// ["これは一行目です", "これは二行目です", "これは三行目です", "これは四行目です"]
```

このコード行は、JavaScript において配列から「falsy」(偽と評価される) 値を取り除く簡潔な方法を示しています。ここで使用されている .filter() メソッドは、配列の各要素に対してテスト関数を実行し、その関数が true を返した要素のみからなる新しい配列を生成します。

## 2. Object のキーを Array にする

```javascript
const users = {
  taro: {
    gender: "男",
    hobby: "水泳",
  },
  jiro: {
    gender: "男",
    hobby: "読書",
  },
  hanako: {
    gender: "女",
    hobby: "カラオケ",
  },
};

Object.keys(users);
// output: ["taro", "jiro", "hanako"]
```

## 3. Object の配列から特定のキーを抽出する

```javascript:example.js
const items = [
 { id: 1, name: "アイテム1" },
 { id: 2, name: "アイテム2" },
 { id: 3, name: "アイテム3" },
];

items.map((item) => item.name);
// output: ["アイテム1", "アイテム2", "アイテム3"]
```

**📌 ポイント**

> - map() メソッドは、配列の各要素に対して与えられた関数を実行し、その結果からなる新しい配列を生成します。
> - ` (item) => item.name` は各オブジェクトの name プロパティの値を返します。

## 4. Object は空か判定する

```typescript:example.ts
const obj = {};

console.log(if(obj));
// true

if (Object.keys(obj).length === 0) {
  console.log("空");
} else {
  console.log("中身あり");
}
```

**📌 ポイント**

> - if(空の object)は true が返却されるので判定できません
> - キーの数で空かどうかを判断する

## 5. `shift()` と `pop()`

```typescript:example.ts
const numbers = [1, 2, 3, 4, 5];
const firstElement = numbers.shift();
const lastElement = numbers.pop();

console.log(firstElement);  // 出力: 1
console.log(numbers);       // 出力: [2, 3, 4, 5]

console.log(lastElement);  // 出力: 5
console.log(numbers);       // 出力: [2, 3, 4]
```

**📌 ポイント**

> - `shift()` メソッドは、配列の先頭から要素を削除し、その要素を返します
> - `pop()` メソッドは、配列の末尾から要素を削除し、その要素を返します
