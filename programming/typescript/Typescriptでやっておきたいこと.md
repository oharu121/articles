## Typescript でやっておきたいこと

## 1. 型アサーション

型アサーションは、開発者が TypeScript の型チェッカーに対して、「私はこの値の型を知っている、そしてそれを特定の型として扱いたい」と伝える方法です。

```diff ts
(async () => {
+  type User = {
+    name: string;
+    age: number;
+  };

  const result = await fetch("www.example.com/getUser/taro");
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

-  const user = await result.json();
+  const user = (await result.json()) as User;

  console.log(`こんにちは、私の名前は${user.name}です。${user.age}歳です。`);
})();
```

## 2. JSDoc を作る

JSDoc を使用すると、関数、クラス、メソッド、パラメータ、戻り値などに関する情報をコメントとして記述できます。これにより、コード自体がその動作や使用方法を説明するドキュメントとして機能し、後からコードを読む人が理解しやすくなります。

**type**

TypeScript では型エイリアスやインターフェースを定義する際に JSDoc コメントを使用できます。これは、その型が何を表しているのか、どのような場合に使用されるのかを説明するのに役立ちます。

```typescript:example.d.ts
/**
 * ユーザーを表す型。
 * @typedef {Object} User
 * @property {number} id ユーザーの一意の識別子。
 * @property {string} name ユーザーのフルネーム。
 * @property {string} email ユーザーのメールアドレス。
 */
export type User = {
  id: number;
  name: string;
  email: string;
};
```

**function**

関数に JSDoc コメントを付けることで、その関数が何をするのか、どのようなパラメータを取り、何を返すのかを明確にすることができます。

```typescript:example.ts
/**
 * 二つの数値の合計を計算します。
 * @param {number} a 最初の数値。
 * @param {number} b 二番目の数値。
 * @returns {number} 二つの数値の合計。
 */
const add = (x: number, y: number): number => {
  return x + y;
};
```

**class**

クラスとそのメソッドに JSDoc コメントを付けることで、クラスの目的、使用方法、各メソッドの機能を文書化できます。

```typescript:example.ts
/**
 * Personクラスは人物を表します。
 */
class Person {
  /**
   * Personのインスタンスを作成します。
   * @param {string} name 人物の名前。
   * @param {number} age 人物の年齢。
   */
  constructor(name, age) {
    /** @type {string} */
    this.name = name;
    /** @type {number} */
    this.age = age;
  }

  /**
   * 人物の自己紹介をします。
   * @returns {string} 自己紹介の文。
   */
  introduce() {
    return `こんにちは、私の名前は${this.name}です。${this.age}歳です。`;
  }
}
```

**✨ 効果**

関数にカーソルを合わせると説明文が出ます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/95e20174-d500-8a13-bc53-bb5673916890.png)

## 3. import 先を@で指定する

エイリアスを使用しない場合、ファイルのインポートには相対パスを使用する必要があります。これは、特にディレクトリ構造が深い場合に、長く複雑なパスになることがあります。

```typescript:example.ts
// src/components/userComponent.ts から src/models/User.ts をインポート
import { User } from '../../models/User';
```

そこで `tsconfig.json` でエイリアスを指定できます

```diff ts:tsconfig.json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
+    "paths": {
+      "@*": ["./*"]
+    },
    "rootDir": "./src",
    "outDir": "./dist",
    "esModuleInterop": true,
    "strict": true
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}
```

エイリアスを使用することで、インポートパスが簡潔で読みやすくなり、ディレクトリ構造が変更された場合でもインポートステートメントを容易に管理できるようになります。

```diff_typescript:example.ts
// src/components/userComponent.ts から src/models/User.ts をインポート
-import { User } from '../../models/User';
+import { User } from '@models/User';
```

**⚠️ モジュール以外で実装したい場合**

普通のスクリプトでエイリアスを使ったら下記のようなエラーが出ます。

```
Error: Cannot find module '@util/messageUtil'
Require stack:
- E:\Script_testing\qitta\colorful-console\main.ts
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader:1144:15)
    at Function.Module._resolveFilename.sharedData.moduleResolveFilenameHook.installedValue [as _resolveFilename] (E:\Script_testing\qitta\colorful-console\node_modules\@cspotcode\source-map-support\source-map-support.js:811:30)
    at Function.Module._load (node:internal/modules/cjs/loader:985:27)
    at Module.require (node:internal/modules/cjs/loader:1235:19)
    at require (node:internal/modules/helpers:176:18)
    at Object.<anonymous> (E:\Script_testing\qitta\colorful-console\main.ts:1:1)
    at Module._compile (node:internal/modules/cjs/loader:1376:14)
    at Module.m._compile (E:\Script_testing\qitta\colorful-console\node_modules\ts-node\src\index.ts:1618:23)
    at Module._extensions..js (node:internal/modules/cjs/loader:1435:10)
    at Object.require.extensions.<computed> [as .ts] (E:\Script_testing\qitta\colorful-console\node_modules\ts-node\src\index.ts:1621:12) {
  code: 'MODULE_NOT_FOUND',
  requireStack: [ 'E:\\Script_testing\\qitta\\colorful-console\\main.ts' ]
}
```

**1. module-alias をダウンロード**

```
npm install module-alias --save-dev
```

**2. package.json にパスを追加する**

```diff js:package.json
{
  "name": "example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
+ "_moduleAliases": {
+   "@util": "util"
+ },
  "scripts": {
    "main": "ts-node main.ts"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "dayjs": "^1.11.10"
  },
  "devDependencies": {
    "@types/node": "^20.12.5",
    "module-alias": "^2.2.3",
    "ts-node": "^10.9.2",
    "typescript": "^5.4.4"
  }
}
```

**3. 実行したいスクリプトにインポートを追加する**

```diff_typescript:main.ts
+import "module-alias/register"
import * as MU from "@util/messageUtil"

MU.info("infoです");
MU.task("taskです");
MU.warning("warningです");
MU.error("errorです");
```

## 4. 複雑な type を分解する

型を分解して定義することのメリットは以下の通りです：

> 1.  **可読性の向上**: 型が単純化され、各型の責務が明確になるため、コードの読みやすさが向上します。
> 2.  **再利用性の向上**:分解された type は他のコンテキストでも再利用しやすくなります。
> 3.  **メンテナンスの容易さ**: 型が分割されていると、特定の部分の変更が他の部分に影響を与えにくくなり、メンテナンスが容易になります。

以下の例を考えてください。どう type を定義しますか？

```typescript:example.ts
const user = {
  id: 1,
  name: "Alice Johnson",
  contact:{
    phone: "080-8466-5622",
    email: "alice.johnson@example.com",
  },
  posts: [
    {
      id: 101,
      title: "My First Post",
      content: "This is the content of my first post.",
      comments: [
        {
          id: 1001,
          content: "Great post!",
          author: "Bob Smith",
        },
        {
          id: 1002,
          content: "Thanks for sharing.",
          author: "Charlie Davis",
        },
      ],
    },
  ],
};
```

下記に定義してみたところ、読みにくいですね。

```typescript:types.d.ts
type User = {
  id: number;
  name: string;
  contact: {
    phone: string;
    email: string;
  };
  posts: {
    id: number;
    title: string;
    content: string;
    comments: {
      id: number;
      content: string;
      author: string;
    }[];
  }[];
};
```

分解して定義すれば読みやすくなります。

```typescript:types.d.ts
type User = {
  id: number;
  name: string;
  contact: Contact; // Contact typeを利用する
  posts: Post[]; // Post typeを利用する
};

type Contact = {
  phone: string;
  email: string;
};

type Post = {
  id: number;
  title: string;
  content: string;
  comments: Comment[]; // Comment typeを利用する
};

type Comment = {
  id: number;
  content: string;
  author: string;
};
```

## 5. 関数に型をつける

関数のパラメータと戻り値に型を指定することで、TypeScript のコンパイラは型の不一致や潜在的なエラーをコードが実行される前に検出できます。これにより、ランタイムエラーのリスクが減少し、デバッグ時間が短縮されます。

下記関数 add は number type の引数 x, y を受け入れて、number type の値を返却します。

```typescript:example.ts
const add = (x: number, y: number): number => {
  return x + y;
};

const result = add(5, 10); // 15
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/7364a72c-f9a9-1771-bcaa-34a248deb97f.png)

> add に string を渡すとエラーが出ます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/dd0b37a4-a066-e595-3dfc-29b6531c284b.png)

> また返却値を string として扱うとエラーが出ます。
