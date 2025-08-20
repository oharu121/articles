## 「push()」は NG？React で ToDo リストを作る前に知っておきたい配列更新の鉄則

## はじめに

React で ToDo リストを実装する際、よく直面するのが「**配列の状態をどう更新するか**」という問題です。

実はこれ、**React 固有の話ではなく、JavaScript の配列操作**にすべてがかかっています。だからこそ、しっかり理解すれば他の JavaScript プロジェクトにも応用可能です。

この記事では、以下の 3 つの操作に分けて、React における配列 state の正しい更新方法を解説します：

- ✅ 要素の追加
- ❌ 要素の削除
- 🔁 要素の更新

## ✅ ToDo の追加：`...` スプレッド構文

配列に新しい要素を追加するには、元の配列を変更せずに、スプレッド構文で新しい配列を作ります。

```tsx
const handleAddTodo = (newTodo) => {
  setTodos([...todos, newTodo]);
};
```

例：

```tsx
handleAddTodo({ id: 3, text: "ブログを書く", done: false });
```

これで `todos` に新しいタスクが追加されます。

## ❌ ToDo の削除：`filter` メソッド

特定の要素を削除したい場合は、`filter` を使って「**残す要素だけを選び直す**」という考え方で新しい配列を作ります。

```tsx
const handleDeleteTodo = (id) => {
  setTodos(todos.filter((todo) => todo.id !== id));
};
```

例：

```tsx
handleDeleteTodo(2); // idが2のToDoを削除
```

## 🔁 ToDo の更新：`map` メソッド

既存の要素の一部（たとえば `done` 状態）を更新したい場合は、`map` を使って「**特定の要素だけ差し替えた新しい配列**」を作ります。

```tsx
const handleToggleTodo = (id) => {
  const newTodos = todos.map((todo) =>
    todo.id === id ? { ...todo, done: !todo.done } : todo
  );
  setTodos(newTodos);
};
```

## ポイント： 「直接変更しない」ことが鉄則

すべてのケースに共通するポイントは：

- **元の配列を直接変更せず、新しい配列を作る**
- **変更には`setState`を使う**
- **JavaScript の基本的な配列操作で十分対応可能**

React の state 更新でバグを起こしやすいのは、**破壊的変更（mutate）をしてしまうケース**です。

```tsx
// ❌ NG例
todos.push(newTodo);
setTodos(todos); // これはバグの元
```

## ✨ おわりに

ToDo リストの管理は React 初心者の登竜門ですが、「**配列を安全に更新する習慣**」を身につける絶好の練習にもなります。

| 操作 | 方法                                       |
| ---- | ------------------------------------------ |
| 追加 | `setTodos([...todos, newTodo])`            |
| 削除 | `setTodos(todos.filter(t => t.id !== id))` |
| 更新 | `setTodos(todos.map(...))`                 |

配列 state を扱う時は、 **必ず「新しい配列を作る」** ことを意識しましょう！
