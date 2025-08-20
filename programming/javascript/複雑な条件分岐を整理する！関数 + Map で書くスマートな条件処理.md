## 複雑な条件分岐を整理する！関数 + Map で書くスマートな条件処理

## はじめに

JavaScript で複雑な条件分岐を記述する場合、`switch`文や`if-else`を使うのが一般的です。しかし、**各条件に複数行の処理や一時変数が含まれる場合、コードが非常に読みにくくなります**。

そこで本記事では、**関数と`Map`を組み合わせて条件ごとの処理を整理するテクニック**を紹介します。各処理を独立した関数に分けることで、読みやすさ・保守性が格段に向上します。

## 読みにくい典型例：switch 文の肥大化

```javascript
function handleEvent(eventType, data) {
  switch (eventType) {
    case "login":
      const username = data.username;
      const time = new Date();
      console.log(`[LOGIN] ${username} at ${time.toISOString()}`);
      break;
    case "logout":
      const logoutTime = new Date();
      console.log(
        `[LOGOUT] User ${
          data.username
        } logged out at ${logoutTime.toISOString()}`
      );
      break;
    case "error":
      console.error(`[ERROR] Code: ${data.code}, Message: ${data.message}`);
      break;
    default:
      console.warn("Unknown event type:", eventType);
  }
}
```

**問題点：**

- 条件ごとの処理が長くなると読みづらい
- `case` ごとに変数を宣言するとスコープが不明瞭
- ロジック変更のたびに`switch`全体を見直す必要がある

## 解決策：関数 + Map による整理

各条件を**独立した関数**にし、それを`Map`で管理することで、コードをシンプルかつ柔軟に記述できます。

#### 1. 関数に切り出す

```javascript
function handleLogin(data) {
  const username = data.username;
  const time = new Date();
  console.log(`[LOGIN] ${username} at ${time.toISOString()}`);
}

function handleLogout(data) {
  const logoutTime = new Date();
  console.log(
    `[LOGOUT] User ${data.username} logged out at ${logoutTime.toISOString()}`
  );
}

function handleError(data) {
  console.error(`[ERROR] Code: ${data.code}, Message: ${data.message}`);
}
```

#### 2. Map に登録して一元管理

```javascript
const handlers = new Map([
  ["login", handleLogin],
  ["logout", handleLogout],
  ["error", handleError],
]);
```

#### 3. Map から関数を呼び出す

```javascript
function handleEvent(eventType, data) {
  const handler = handlers.get(eventType);
  if (handler) {
    handler(data);
  } else {
    console.warn("Unknown event type:", eventType);
  }
}
```

**メリットまとめ**

| 項目           | 関数＋ Map                     | switch/if                          |
| -------------- | ------------------------------ | ---------------------------------- |
| 処理の見通し   | 各処理が独立していて読みやすい | 複数行になると見通しが悪い         |
| 保守性         | 関数単位で変更しやすい         | 巨大な switch ブロックが壊れやすい |
| テストしやすさ | 各関数を個別にテスト可能       | case ごとに切り出しづらい          |
| スコープ管理   | 関数スコープ内で完結           | ブロックスコープが曖昧になりやすい |

## まとめ

`switch`や`if`は簡単な条件分岐には便利ですが、**処理が複雑になると保守性や可読性に大きな影響を与えます**。

- 複数行の処理や変数宣言が必要な場合
- 処理をテスト・再利用したい場合
- 条件分岐が動的に変わる場合

このようなケースでは、\*\*関数と Map を活用することで、ロジックをスッキリ整理することができます。
