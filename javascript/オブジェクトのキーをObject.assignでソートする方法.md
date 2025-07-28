# はじめに

JavaScriptでオブジェクトを扱う際、オブジェクトのキーを特定の順序で整理したい場合があります。例えば、オブジェクトのデータをCSVに出力する際、特定の順序でキーを並べたいことがあります。この記事では、Object.assignを使用してオブジェクトのキーをソートする方法について解説します。

次のような場面で困ったことはありませんか？

```typescript:example.ts
import fs from "fs";

const data = await fetch("www.foo.com");
// dataの中身 = { age: 18, hobby: "basketball", name: "太郎" }

const headers = Object.keys(data); // ヘッダーを取得する
const values = Object.values(data); // 値を取得する

fs.appendFileSync("example.csv", headers.join(","));
fs.appendFileSync("example.csv", "\n");
fs.appendFileSync("example.csv", values.join(","));
```
このコードの実行結果として出力されるexample.csvは次のようになります：

```example.csv
age,hobby,name
18,basketball,太郎
```

しかし、以下のような順序で出力したい場合、どうすればよいでしょうか？

```example.csv
name,age,hobby
太郎,18,basketball
```

# Object.assignとは？

まず、Object.assignの基本的な使い方を確認しましょう。Object.assignは、複数のソースオブジェクトのプロパティをターゲットオブジェクトにコピーするためのメソッドです。

```javascript
const target = {};
const source = { a: 1, b: 2 };
Object.assign(target, source);
console.log(target); // { a: 1, b: 2 }
```

この例では、sourceオブジェクトのプロパティがtargetオブジェクトにコピーされました。

# オブジェクトのキーをソートする

オブジェクトのキーを特定の順序でソートするには、あらかじめキーの順序を定義したオブジェクトを用意し、それをObject.assignでマージする方法があります。以下に、その実装例を示します。

```diff typescript:example.ts
import fs from "fs";

const data = await fetch("www.foo.com");
// dataの中身 = { age: 18, hobby: "basketball", name: "太郎" }

+ // キーの順序を定義するテンプレートオブジェクト
+ const objOrder = {
+   name: "",
+   age: "",
+   hobby: ""
+ }

+ // テンプレートに従ってソートされたオブジェクトを作成
+ const sortedObj = Object.assign(objOrder, data);
- const headers = Object.keys(data); // ヘッダーを取得する
+ const headers = Object.keys(sortedObj); // ヘッダーを取得する
- const values = Object.values(data); // 値を取得する
+ const values = Object.values(sortedObj); // 値を取得する

fs.appendFileSync("example.csv", headers.join(","));
fs.appendFileSync("example.csv", "\n");
fs.appendFileSync("example.csv", values.join(","));
```

このコードを実行すると、次のようなexample.csvが出力されます：

```example.csv
name,age,hobby
太郎,18,basketball
```

# 追記：スレッド構文

credit to: @shiracamus さん
Object.assignのほかに、スレッド構文も同じ効果があります。
```typescript
const sortedObj = { ...objOrder, ...data };
```

# 注意点

1. ネストされたオブジェクトにむいていません
1. 大規模なオブジェクトで使用する場合、パフォーマンスに注意が必要です

# まとめ

この記事では、Object.assignを使用してオブジェクトのキーをソートする方法を紹介しました。APIレスポンスを整形したり、キーの順序を一貫性を持たせたい場合に便利です。ぜひ、実際のプロジェクトで活用してみてください。
