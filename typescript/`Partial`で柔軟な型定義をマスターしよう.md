# はじめに

TypeScriptで開発していると、こんな場面はありませんか？

>* ユーザー情報のうち、**名前だけを更新する関数**を作りたい
>* フォームの入力項目が多く、**必須項目を段階的に埋めていく**処理を書きたい
>* テストデータを作りたいけど、**毎回すべてのプロパティを定義するのが面倒**…

こんな「**型の一部分だけを扱いたい**」という悩みを華麗に解決してくれるのが、今回紹介する`Partial<T>`です。

この記事では、`Partial<T>`の基本的な使い方から、実務で役立つ活用パターン、そして思わぬ落とし穴まで、分かりやすく解説します。

# そもそも `Partial<T>`とは？

`Partial<T>`は、TypeScriptが標準で提供している「ユーティリティ型」の一つです。

一言でいうと、**既存の型のすべてのプロパティを「あってもなくても良い（オプショナル `?`）」状態に変換してくれる魔法のような型**です。

百聞は一見に如かず、コードで見てみましょう。

```typescript
// 元となるUser型
type User = {
  id: number;
  name: string;
  email: string;
};

// User型をPartialで変換！
type PartialUser = Partial<User>;

// ↓↓↓ PartialUser は、下の型と全く同じ意味になります ↓↓↓

type PartialUser = {
  id?: number;      // '?' がついてオプショナルに！
  name?: string;    // '?' がついてオプショナルに！
  email?: string;   // '?' がついてオプショナルに！
};
```

`Partial<User>`と書くだけで、`id`、`name`、`email`がすべてオプショナルな型を簡単に作り出すことができました。

# どんな時に便利？ `Partial<T>`の活用パターン3選

`Partial<T>`が特に輝く、代表的な3つのシナリオを紹介します。

### 1\. オブジェクトの一部だけを更新する関数に

ユーザー情報の一部だけを更新する関数を例に考えてみましょう。`Partial<T>`がない場合、更新用の型を別途定義する必要があり面倒です。

```typescript
// ✅ Partialを引数の型に使う
function updateUser(user: User, updates: Partial<User>): User {
  // スプレッド構文で元のuserオブジェクトに更新内容（updates）をマージ
  return { ...user, ...updates };
}

// --- 実行例 ---
const currentUser: User = {
  id: 1,
  name: "山田太郎",
  email: "taro@example.com",
};

// nameだけを更新したい場合、updatesにはnameだけ渡せばOK！
const updatedUser = updateUser(currentUser, { name: "山田花子" });

console.log(updatedUser);
// { id: 1, name: "山田花子", email: "taro@example.com" }
```

>引数の型を`Partial<User>`にすることで、`id`, `name`, `email` のうち、**更新したいプロパティだけを持つオブジェクトを安全に受け取る**ことができます。

### 2\. フォーム入力など、段階的に作られるデータに

複数のステップに分かれたフォームや、ユーザーが項目を少しずつ埋めていくようなUIでは、最初からすべてのデータが揃っているわけではありません。

`Partial<T>`を使えば、**入力途中の不完全な状態のオブジェクト**を型安全に扱うことができます。

```typescript
type UserProfile = {
  name: string;
  age: number;
  address: string;
};

// 入力途中のフォームデータをPartialで定義
let formState: Partial<UserProfile> = {};

// ユーザーが名前を入力した
formState.name = "鈴木一郎";

// 次に住所を入力した
formState.address = "東京都...";

// この時点では age は undefined だが、型エラーにはならない
```

### 3\. テスト用のモックデータを柔軟に作成

テストコードを書く際、毎回すべてのプロパティを律儀にセットアップするのは大変です。`Partial<T>`を使えば、**デフォルト値を持ちつつ、テストごとに必要な値だけを上書きする**スマートな関数が作れます。

```typescript
function createUserMock(overrides: Partial<User> = {}): User {
  const defaultUser: User = {
    id: 1,
    name: "Default User",
    email: "default@example.com",
  };

  return { ...defaultUser, ...overrides };
}

// --- 実行例 ---

// デフォルト値のままユーザーを作成
const user1 = createUserMock();
// { id: 1, name: "Default User", ... }

// nameだけ指定して、他はデフォルト値を使いたい
const user2 = createUserMock({ name: "Test User" });
// { id: 1, name: "Test User", ... }

// emailとidを上書き
const user3 = createUserMock({ id: 99, email: "test@test.com" });
// { id: 99, name: "Default User", email: "test@test.com" }
```

# ⚠️ 注意点

`Partial<T>`は非常に便利ですが、一つだけ注意すべき大きなポイントがあります。それは、**プロパティが`undefined`かもしれない**ということです。

すべてのプロパティがオプショナルになるため、プロパティが存在することを前提としたコードはエラーを引き起こします。

```typescript
// ❌ 悪い例: user.nameが存在するとは限らない！
function getGreeting(user: Partial<User>) {
  // コンパイルエラー: 'user.name' is possibly 'undefined'.
  console.log(`Hello, ${user.name.toUpperCase()}`);
}
```

**対策：プロパティの存在チェックを行う**

この問題を避けるには、プロパティを使用する前に必ず存在を確認（**型ガード**）するか、**デフォルト値**を設定します。

```typescript
// ✅ 良い例
function getGreeting(user: Partial<User>) {
  // 対策1: if文で存在をチェックする
  if (user.name) {
    console.log(`Hello, ${user.name.toUpperCase()}`);
  } else {
    console.log("Hello, Guest!");
  }

  // 対策2: Null合体演算子(??)でデフォルト値を設定する
  const name = user.name ?? "Guest";
  console.log(`Hello, ${name.toUpperCase()}`);
}
```

`Partial<T>`を使う際は、常に「この値は`undefined`かも？」と意識することが大切です。

# 補足：`Partial<T>`を自作して仕組みを理解する

`Partial<T>`は、実はTypeScriptの**Mapped Types**という機能を使ってシンプルに実装できます。その仕組みを知ると、より理解が深まります。

```typescript
// これがPartial<T>の正体！
type MyPartial<T> = {
  [P in keyof T]?: T[P];
};
```

この一行を分解すると、

  * `keyof T`: `T`のすべてのプロパティ名（`'id' | 'name' | 'email'`など）を取り出す。
  * `[P in ... ]`: 取り出したプロパティ名を一つずつ`P`に入れてループさせる。
  * `?:`: プロパティをオプショナルにする。
  * `T[P]`: 元の型`T`が持つプロパティ`P`の型（`number`や`string`など）を参照する。

つまり、「**型`T`のすべてのキーをループ処理して、それぞれをオプショナルなプロパティとして再定義する**」ということを行っているのです。

# おわりに

`Partial<T>`は、TypeScriptにおける「DRY（Don't Repeat Yourself）」の原則を体現したような、非常に強力なユーティリティ型です。

>* **オブジェクトの部分的な更新**
>* **入力途中のデータ表現**
>* **柔軟なテストデータ作成**

これらの場面で`Partial<T>`を使いこなせれば、よりクリーンで保守性の高いコードを書けるようになります。ただし、**`undefined`の可能性** には常に注意を払いましょう。

TypeScriptには他にも`Required<T>`（すべて必須にする）や`Pick<T, K>`（一部だけ抜き出す）など、便利なユーティリティ型がたくさんあります。`Partial<T>`に慣れたら、ぜひ他の型も調べてみてください！
