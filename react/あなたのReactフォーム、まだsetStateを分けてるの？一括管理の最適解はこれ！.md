# はじめに

Reactで会員登録フォームや設定画面などを作成する際、複数の入力項目（input）が登場します。これらの値を効率的に `useState` で管理するには、イベントハンドラである `handleChange` 関数の設計が非常に重要です。

この記事では、**たった一つの`handleChange`関数で、フォーム全体の入力状態を一元管理する**、クリーンで拡張性の高い方法を解説します。

**❌ アンチパターン：inputごとに個別のイベントハンドラを定義する**

フォームの入力欄が増えるたびに、ついやってしまいがちなのが、`useState` を更新する関数を各input専用に作ってしまうことです。

```jsx
const [formData, setFormData] = useState({ name: '', email: '' });

// 👎 name用のハンドラ
const handleNameChange = (e) => {
  setFormData({ ...formData, name: e.target.value });
};

// 👎 email用のハンドラ
const handleEmailChange = (e) => {
  setFormData({ ...formData, email: e.target.value });
};
```

この方法は直感的ですが、項目が増えるたびに関数を追加する必要があり、コードが冗長になりメンテナンス性も低下します。

# 解決策：共通の`handleChange`関数

この問題を解決するのが、`input` タグの **`name` 属性** を活用した、共通の `handleChange` 関数です。

```javascript
const handleChange = (e) => {
  setFormData(prevFormData => ({
    ...prevFormData,
    [e.target.name]: e.target.value
  }));
};
```

この関数が一つあれば、フォームにいくらinputが増えても対応できます。仕組みは以下の通りです。

>1.  **`...prevFormData`**
オブジェクトのスプレッド構文です。これにより、`name` や `email` など、状態オブジェクト（`formData`）に含まれる既存の値をすべてコピーします。これをしないと、更新対象以外の値が消えてしまいます。

>2.  **`[e.target.name]`**
JavaScriptの**Computed Property Names**という構文です。`[]`でキーを囲むことで、変数や式の評価結果をオブジェクトのキーとして動的に設定できます。
      * `e.target` はイベントを発生させた要素（この場合は `<input>`）を指します。
      * `e.target.name` はその要素の `name` 属性の値（例: "name", "email"）を返します。

>3.  **`e.target.value`**
    入力された新しい値を指します。

つまり、この一行は「**`formData` の中身はそのままに、変更があったinputの`name`属性と一致するキーの値を、新しい`value`で更新する**」という意味になります。

# 使用例：マルチステップフォーム

このテクニックが実際にどのように機能するか、複数の入力項目を持つフォームで見てみましょう。

```jsx:Multistepform.jsx
import { useState } from 'react';

const initialFormData = {
  name: '',
  email: '',
  password: '',
};

export default function MultiStepForm() {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState(initialFormData);

  // ✨ たった一つの関数で全てのinputに対応！
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prevFormData => ({
      ...prevFormData,
      [name]: value,
    }));
  };

  const nextStep = () => setStep((prev) => prev + 1);
  const prevStep = () => setStep((prev) => prev - 1);

  return (
    <div>
      {step === 1 && (
        <div>
          <h2>Step 1: 名前</h2>
          <input
            type="text"
            name="name" // ← このname属性が [e.target.name] に対応
            value={formData.name}
            onChange={handleChange}
            placeholder="Your Name"
          />
          <button onClick={nextStep}>次へ</button>
        </div>
      )}
      {step === 2 && (
        <div>
          <h2>Step 2: メールアドレス</h2>
          <input
            type="email"
            name="email" // ← このname属性が [e.target.name] に対応
            value={formData.email}
            onChange={handleChange}
            placeholder="Your Email"
          />
          <button onClick={prevStep}>戻る</button>
          <button onClick={nextStep}>次へ</button>
        </div>
      )}
      {step === 3 && (
        <div>
          <h2>Step 3: パスワード</h2>
          <input
            type="password"
            name="password" // ← このname属性が [e.target.name] に対応
            value={formData.password}
            onChange={handleChange}
            placeholder="Your Password"
          />
          <button onClick={prevStep}>戻る</button>
          <button onClick={() => alert(JSON.stringify(formData, null, 2))}>
            送信
          </button>
        </div>
      )}
    </div>
  );
}
```

# 応用編：チェックボックスやその他の要素を扱う

テキスト入力だけでなく、チェックボックスのように値の取得方法が異なる要素も扱いたい場合があります。その場合は、`type` プロパティに応じて処理を分岐させると、同じ関数で対応できます。

```javascript
const handleChange = (e) => {
  const { name, value, type, checked } = e.target;
  
  setFormData(prevFormData => ({
    ...prevFormData,
    [name]: type === 'checkbox' ? checked : value
  }));
};
```

>* **`type === 'checkbox'`**: inputのタイプがチェックボックスの場合、`value` ではなく `checked` プロパティ（`true` or `false`）を状態として保存します。
>* それ以外（`text`, `email`, `password`, `select-one` など）の場合は、これまで通り `value` を保存します。

# まとめ

>* **`input` の `name` 属性**を活用し、**共通の `handleChange` 関数**を作成する。
>* **`[e.target.name]`** のようにComputed Property Namesを使うことで、どの状態を更新するかを動的に決定できる。
>* このアプローチにより、コードの**冗長性を排除**し、**保守性と拡張性**が劇的に向上する。

このテクニックは、Reactでのフォーム開発における基本かつ非常に強力なパターンです。ぜひマスターして、クリーンなコードを目指しましょう！