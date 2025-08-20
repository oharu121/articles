## はじめに

Node.js のコードを眺めていると、`!!token` のような一見不思議な記述を目にすることがあります。
「なぜビックリマークが2つも続くの？」と感じた方も多いのではないでしょうか。
実はこの `!!` は、**任意の値を真偽値（boolean）に変換するためのシンプルかつ短い書き方**です。
本記事では、この記法の意味や使いどころ、そして注意点について解説します。

## `!!` の基本的な意味

>* `!` は「否定（NOT）」を意味する論理演算子

```js
!true   // false
!false  // true
!'abc'  // false（truthyな値）
!0      // true（falsyな値）
```

>* `!!` は「2回否定」することで元の真偽値に戻す
  → **boolean型にキャストする効果**

```js
!!'abc' // true
!!0     // false
!!null  // false
!!{}    // true
```

**なぜ `!!` を使うのか？**

>* JavaScriptは `if (token)` のように書くと truthy/falsy 判定してくれるが、変数に明確に `true` / `false` を代入したい場面では `!!` が便利
>* 明示的に boolean に変換した方が意図が伝わることも多い
>* 型が重要な TypeScript や JSON API のレスポンス整形でも役立つ

## Node.js 実務での利用例

### 1\. 環境変数の有無を判定

```js
const hasToken = !!process.env.API_TOKEN;
console.log(hasToken); // true または false
```

**📌 ポイント**

>* `process.env` はすべて **文字列または undefined**
>* `!!` を使うことで明示的に boolean に変換

### 2\. ユーザー認証チェック

```js
function isAuthenticated(req) {
  return !!req.headers.authorization;
}

if (isAuthenticated(req)) {
  // 認証済みの処理
} else {
  // 認証エラー
}
```

**📌 ポイント**

>* `!!` によって `authorization` ヘッダーが存在するかを簡潔に判定

### 3\. データ整形

```js
const user = {
  id: 1,
  name: 'Alice',
  isActive: !!row.active_flag // DBの数値をboolean化
};
```

**📌 ポイント**

>* MySQLやPostgreSQLの `0` / `1` フラグを boolean に変換

**🌐 似たような方法との比較**

| 方法                     | 説明          | 例                         |
| ---------------------- | ----------- | ------------------------- |
| `!!value`              | 最短でboolean化 | `!!'abc'` → `true`        |
| `Boolean(value)`       | 明示的で可読性高め   | `Boolean('abc')` → `true` |
| `value ? true : false` | 三項演算子で変換    | 長くなるが条件付き処理も可能            |

**‼️ 注意点**

>* `!!` は「truthy/falsy」に依存するため、0 や空文字も false になる
>* 0 を有効な値として扱いたい場合は使わない方がいい

  ```js
  !!0 // false（意図と違うかも）
  ```

## まとめ

>* `!!` は JavaScript で **boolean型に変換する短縮記法**
>* Node.js では環境変数やリクエストヘッダーの存在判定に便利
>* ただし falsy 判定の仕様を理解して使うことが大事
