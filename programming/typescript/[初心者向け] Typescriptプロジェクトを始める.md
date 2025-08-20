## [初心者向け] Typescript プロジェクトを始める

## はじめに

TypeScript を使用することで、型を通じて早期にエラーを発見し、大規模アプリケーションのための強力な機能を活用することができます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/fda2cea0-bfba-8aeb-b486-d2b782dc93cd.png)

## 1. TypeScript プロジェクトの初期化

プロジェクトに package.json ファイルがまだない場合は、新しい Node.js プロジェクトを初期化して始めます：

```
npm init -y
```

次に、TypeScript と ts-node をインストールします。ts-node を通じて TypeScript ファイルを直接実行できます。

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

## 2. TypeScript 設定ファイルの作成

下記コマンドで tsconfig.json を作成します。

```
npx tsc --init
```

下記の内容で tsconfig.json を上書きします。

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

> - `target`：コンパイルされた JavaScript のバージョンを指定する
> - `module`：モジュールコード生成の種類を指定する
> - `outDir`：出力されるファイルのディレクトリを指定する
> - `rootDir`：コンパイルするファイルが含まれるルートディレクトリを指定する
> - `strict`：すべての厳格型チェックを有効にする
> - `esModuleInterop`：CommonJS と ES モジュール間の相互運用性を有効にする
> - `include`：typescript に適用したいファイルのパターンのを指定する
>   "\*\*/_"は、すべてのファイルを typescript に適用する
>   "\*\*/_.ts"は、TypeScript ファイルのみ適用する
>   "src/\*_/_"は、src ディレクトリ配下のものをすべて適用する
> - `exclude`：typescript 適用から除外したいファイルのパターンのを指定する
>   node_modules は基本的提供したくないので除外する

## 3. ライブラリの Type 補完ファイル

ファイルの名前を変更した後、型が不足しているなどの理由で TypeScript のエラーが発生する可能性があります。type ファイルもインストールしましょう！

```
npm install @types/lodash --save-dev
```

## 4. 実行およびビルドスクリプト

typescript ファイルを実行するには下記の行をスクリプトに追加します。

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

これで「npm run foo」で foo.ts を実行できます。

```
npm run foo
```

## 補足：JavaScript から移行

**1. 拡張子を変更する**

.js ファイルの拡張子を.ts に変更します。プロジェクトで JSX を使用している場合は、.jsx ファイルの拡張子を.tsx に変更します。

**2. インポートステートメントを調整する**

JavaScript (JS) から TypeScript (TS) への移行時には、特に CommonJS スタイルの require() から ES モジュール スタイルの import ステートメントに移行する場合、インポートステートメントを調整する必要があります。

```diff_typescript:foo.ts
-const express = require('express');
-const { myFunction } = require('./myModule');
+import express from 'express';
+import { myFunction } from './myModule';
```

## まとめ

この記事では、TypeScript を使用して JavaScript プロジェクトを開始または移行するための基本的なステップを紹介しました。TypeScript の導入は、型を通じてコードの信頼性を高め、開発プロセスを改善するための効果的な手段です。
