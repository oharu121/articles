# はじめに

JavaScriptにおける `||`（論理和）と `&&`（論理積）、そして `??`（null合体演算子）は、**条件分岐**や**値の選択処理**で非常に便利な演算子です。ただの「真偽値チェック」ではなく、**値の返し方に特徴がある**点が重要です。

本記事では以下の内容を解説します：

    * `||` / `&&` / `??` の違い
* よくある活用パターン（if なしで書ける！）
* React での使用例
* 使いこなしのコツと注意点

# `||` 、 `&&` 、 `??` の基本動作

JavaScriptでは、`||` 、 `&&` 、 `??` は **条件に応じてその値自体を返す**のが特徴です。

```js
console.log(false || 'default') // 左が falsy なら右を返す → 'default'
console.log(null ?? 'default') // 左が null/undefined なら右を返す → 'default'
console.log('hello' && 123)   // 左が truthy なら右を返す → 123
```

# 活用例①：デフォルト値の設定（`||`）

```js
const name = inputName || 'ゲスト';
```

`inputName` が空文字・null・undefined・0 などの falsy 値なら `'ゲスト'` が代入されます。

```js
const count = inputCount ?? 0;
```

* `inputCount` が `null` または `undefined` のときだけ `0` を使う
* `0` や `''` を許容したい場合は `??` を使うのがベスト

```js
const password = process.env.PASSWORD || "";
```
Node.jsなどで環境変数を使う場合、`process.env.XXX` は `string | undefined` になります。

明示的に `string` として扱いたいときに `|| ""` を使うと便利です。
* 型エラー回避
* `.trim()` などの文字列メソッドを安心して使えるようになります

# 活用例②：条件付きで処理実行（`&&`）

```js
isLoggedIn && showDashboard();
```

`isLoggedIn` が true の場合のみ `showDashboard()` を実行します。

```js
// 通常のif
if (user) {
  console.log(user.name);
}

// &&で簡潔に
user && console.log(user.name);
```
ifよりシンプルに書けます。

# 活用例③：複数条件と組み合わせる

```js
const msg = (isAdmin && '管理者') || '一般ユーザー';
```

→ `isAdmin` が true なら `'管理者'`、false なら `'一般ユーザー'` が代入されます。

# 補足：truthy / falsy を理解しておく

`||` は falsy 値すべてを無視します：

```js
false
0
''
null
undefined
NaN
```

一方 `??` は `null` と `undefined` だけを無視します。

```js
0 ?? 100 // → 0
0 || 100 // → 100
```

つまり、**`0` や `''` を有効な値として使いたいときは `??` を使うべき**です。

# おまけ：Reactでの活用

```jsx
{isLoading && <Spinner />}
{error && <ErrorMessage message={error} />}
```

JSX内でも `&&` はよく使われます。これも「trueなら右を評価」の応用ですね。

# おわりに

`||`, `&&`, `??` を理解し活用できるようになると、**if文を減らし、より宣言的で読みやすいコードが書けるようになります。**

* falsy 値か null/undefined かで適切な演算子を選ぶ
* JSX でも自然に使える
* 型安全にも貢献（`|| ""` で `string` に変換など）

ぜひ、日々のコーディングに取り入れてみてください💡
