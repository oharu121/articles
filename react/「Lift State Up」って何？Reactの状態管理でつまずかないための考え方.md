# はじめに

Reactでアプリを作っていると、次のような状況に出会ったことはありませんか？

* 複数のコンポーネント間で同じ状態を共有したい
* 子コンポーネントから親の状態を変更したい
* 状態が増えてコードがごちゃごちゃになってきた...

そんなときに登場するのが「**状態のリフトアップ（Lift State Up）**」という概念です。

# シナリオ１：単一のTodoを表示する😀

まずは単一のTodoコンポーネントに状態を持たせてみます。

```jsx:Todo.jsx
import { useState } from "react";

export default function Todo() {
  const [todo, setTodo] = useState({
    id: 1,
    label: "Learn React",
    completed: false
  });

  const toggleCompleted = () => {
    setTodo({ ...todo, completed: !todo.completed });
  };

  return (
    <label>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={toggleCompleted}
      />
      {todo.label}
    </label>
  );
}
```

これはシンプルで良さそうに見えますが、**複数のTodoを表示したいとき**に困ったことが起きます。

# シナリオ２：複数のTodoを表示する😓

Todoを複数表示したいとします。

```jsx
<Todo />
<Todo />
```

こうすると：

* 同じ初期状態のTodoが並ぶだけで個別に管理できない
* ユーザーが新しいTodoを追加しても反映できない
* 状態が分散して管理しづらい

ここで必要になるのが、「状態を親に上げる（Lift State Up）」です。

# 解決策：状態のリフトアップで解決！

### ステップ１：状態を子 → 親に移動する

```jsx:Todo.jsx
import { useState } from "react";
import Todo from "./Todo";

export default function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, label: "Learn React", completed: false },
    { id: 2, label: "Learn Next.js", completed: false }
  ]);

  const handleUpdateTodo = (updatedTodo) => {
    setTodos(todos.map(todo =>
      todo.id === updatedTodo.id ? updatedTodo : todo
    ));
  };

  return (
    <ul>
      {todos.map(todo => (
        <Todo
          key={todo.id}
          todo={todo}
          onUpdate={handleUpdateTodo}
        />
      ))}
    </ul>
  );
}
```

### ステップ２：親 → 子は props で渡す

```jsx
export default function Todo({ todo, onUpdate }) {
  const toggleCompleted = () => {
    onUpdate({ ...todo, completed: !todo.completed });
  };

  return (
    <li>
      <label>
        <input
          type="checkbox"
          checked={todo.completed}
          onChange={toggleCompleted}
        />
        {todo.label}
      </label>
    </li>
  );
}
```

# Lift State Up のルール

> **「複数のコンポーネントで共有したい状態」は共通の親に持たせる**

その上で、

* 子 → 親への通信は「関数（コールバック）」を props で渡す
* 親 → 子へは「状態」そのものを props で渡す

この構成にすることで、アプリ全体の状態を**一元管理**でき、コードも明快になります。

# 補足：このあと学ぶと良いこと

* `useReducer` を使った状態管理
* コンポーネント分割とコンテキストAPI
* 外部ライブラリ（Recoil, Zustandなど）による状態管理

# おわりに

「状態を親に上げる（Lift State Up）」はReactの基本中の基本ですが、これを意識するだけでアプリの設計がグッとしやすくなります。

まずは小さなアプリから、「どこに状態を置くべきか」を考えてみましょう！
