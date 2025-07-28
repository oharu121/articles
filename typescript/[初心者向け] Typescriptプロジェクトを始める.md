# はじめに

TypeScriptを使用することで、型を通じて早期にエラーを発見し、大規模アプリケーションのための強力な機能を活用することができます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/fda2cea0-bfba-8aeb-b486-d2b782dc93cd.png)

# 1. TypeScriptプロジェクトの初期化

プロジェクトにpackage.jsonファイルがまだない場合は、新しいNode.jsプロジェクトを初期化して始めます：

```
npm init -y
```

次に、TypeScriptとts-nodeをインストールします。ts-nodeを通じてTypeScriptファイルを直接実行できます。

```
npm install typescript ts-node @types/node --save-dev
```

```package.json
{
  "name": "example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@types/node": "^20.11.19",
    "ts-node": "^10.9.2",
    "typescript": "^5.3.3",
  },
}
```

# 2. TypeScript設定ファイルの作成

下記コマンドでtsconfig.jsonを作成します。

```
npx tsc --init
```

下記の内容でtsconfig.jsonを上書きします。

```tsconfig.json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "rootDir": "./",
    "outDir": "./dist",
    "esModuleInterop": true,
    "strict": true
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}
```

>* `target`：コンパイルされたJavaScriptのバージョンを指定する
>* `module`：モジュールコード生成の種類を指定する
>* `outDir`：出力されるファイルのディレクトリを指定する
>* `rootDir`：コンパイルするファイルが含まれるルートディレクトリを指定する
>* `strict`：すべての厳格型チェックを有効にする
>* `esModuleInterop`：CommonJSとESモジュール間の相互運用性を有効にする
>* `include`：typescriptに適用したいファイルのパターンのを指定する
"\*\*/*"は、すべてのファイルをtypescriptに適用する
"\*\*/*.ts"は、TypeScriptファイルのみ適用する
"src/**/*"は、srcディレクトリ配下のものをすべて適用する
>* `exclude`：typescript適用から除外したいファイルのパターンのを指定する
node_modulesは基本的提供したくないので除外する


# 3. ライブラリのType補完ファイル

ファイルの名前を変更した後、型が不足しているなどの理由でTypeScriptのエラーが発生する可能性があります。typeファイルもインストールしましょう！

```
npm install @types/lodash --save-dev
```

# 4. 実行およびビルドスクリプト

typescriptファイルを実行するには下記の行をスクリプトに追加します。

```diff_javascript:package.json
{
  "name": "example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
-    "test": "echo \"Error: no test specified\" && exit 1"
+    "foo": "ts-node foo.ts"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@types/node": "^20.11.19",
    "ts-node": "^10.9.2",
    "typescript": "^5.3.3",
  },
}
```
これで「npm run foo」でfoo.tsを実行できます。
```
npm run foo
```

# 補足：JavaScriptから移行

**1. 拡張子を変更する**

.jsファイルの拡張子を.tsに変更します。プロジェクトでJSXを使用している場合は、.jsxファイルの拡張子を.tsxに変更します。

**2. インポートステートメントを調整する**

JavaScript (JS) から TypeScript (TS) への移行時には、特に CommonJS スタイルの require() から ES モジュール スタイルの import ステートメントに移行する場合、インポートステートメントを調整する必要があります。

```diff_typescript:foo.ts
-const express = require('express');
-const { myFunction } = require('./myModule');
+import express from 'express';
+import { myFunction } from './myModule';
```

# まとめ

この記事では、TypeScriptを使用してJavaScriptプロジェクトを開始または移行するための基本的なステップを紹介しました。TypeScriptの導入は、型を通じてコードの信頼性を高め、開発プロセスを改善するための効果的な手段です。