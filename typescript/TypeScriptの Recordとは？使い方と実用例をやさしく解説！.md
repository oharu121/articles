# TypeScriptの Recordとは？使い方と実用例をやさしく解説！

# はじめに

TypeScriptを使っていて、こんなコードを見かけたことはありませんか？

```ts
const userStatus: Record<string, boolean> = {
  alice: true,
  bob: false,
}
```

一見すると、普通のオブジェクトと何が違うの？と思うかもしれません。
しかし `Record<K, T>` を使うと、「キーの型」と「値の型」をしっかり定義できるというメリットがあります。

この記事では、TypeScriptのユーティリティ型 Record<> について、以下の流れでわかりやすく解説します。
>*	Recordの基本構文と意味
>*	普通のオブジェクトとの違い
>*	よくある使い方の例
>*	型安全性が高まる場面

# `Record`の基本構文と意味

**Record<Keys, Type>**

>*	`Keys`: キーとして使える型（string や リテラル型など）
>*	`Type`: 値の型

例：ユーザーIDとフラグの対応

```ts
const flags: Record<string, boolean> = {
  user1: true,
  user2: false,
}
```

このように、Record<string, boolean> は
「キーが文字列で、値が真偽値（boolean）」のオブジェクトを意味します。

**📌 普通のオブジェクトとの違い**

以下の2つは見た目は同じですが、型の厳しさが違います。

1. 通常のオブジェクト型

```ts
const status: { [key: string]: boolean } = {
  alice: true,
  bob: false,
}
```

2. Recordを使った型定義

```ts
const status: Record<'alice' | 'bob', boolean> = {
  alice: true,
  bob: false,
}
```

>👉 Record の方は 指定したキー以外は使えません。

```ts
const status: Record<'alice' | 'bob', boolean> = {
  alice: true,
  bob: false,
  charlie: true, // ❌ エラー
}
```

>間違ったキーを使うとエラーになります。

# よくある使い方の例

### 1. ロール別の権限

```ts
type Role = 'admin' | 'user' | 'guest'

const permissions: Record<Role, string[]> = {
  admin: ['read', 'write', 'delete'],
  user: ['read', 'write'],
  guest: ['read'],
}
```

### 2. フロントエンドの表示ラベル

```ts
type Lang = 'ja' | 'en' | 'zh'

const labels: Record<Lang, string> = {
  ja: 'こんにちは',
  en: 'Hello',
  zh: '你好',
}
```

# おわりに


Recordを使うと…
>* 定義漏れやタイポを防げる
>* すべてのキーに対して値が必要という制約をかけられる
>* ユニオン型を強く活かせる

つまり、静的なマップオブジェクトの型をしっかり管理したいときにピッタリです！

Record<> は一見地味ですが、TypeScriptらしい強力な型安全性を手に入れるための武器です。
「どのキーにどんな値があるか」を型で表したいとき、ぜひ積極的に使ってみてください！
