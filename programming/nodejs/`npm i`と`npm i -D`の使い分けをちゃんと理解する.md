## `npm i`と`npm i -D`の使い分けをちゃんと理解する

## はじめに

Node.js プロジェクトでパッケージを追加する際に、よく使う `npm i`（もしくは `npm install`）。しかし、オプションで `-D`（もしくは `--save-dev`）をつける場面もよくあります。

初心者のうちは「とりあえず `-D` をつけておけばいいのでは？」と考えがちですが、**開発環境と本番環境での使い分け**を理解しておくことは非常に重要です。

この記事では、`npm i` と `npm i -D` の違いと、それぞれどんなときに使うべきかをわかりやすく解説します。

**なぜ正しく使い分けるべき？**

> - **本番サイズが小さくなる**：開発用の依存を含めないことで、Docker イメージやサーバー上の容量が抑えられます。
> - **セキュリティリスクの低減**：本番に不要なツール（たとえば ESLint やテストツール）をインストールしないことで、依存関係からくるセキュリティリスクを減らせます。
> - **チームの認識統一**：開発用ツールと本番用ライブラリを明確に分けることで、チーム全体での認識が揃い、トラブルを防げます。

## `npm install` の基本

指定したパッケージを **dependencies（本番環境でも必要な依存）** として追加します。

```bash
npm install package-name
## 省略形
npm i package-name
```

**使うべき場面**

> - 本番環境（production）で必要なパッケージ
> - ライブラリやフレームワークなど、アプリの動作に関わるもの

```bash
npm i express
npm i react
```

## `npm install -D` の基本

指定したパッケージを **devDependencies（開発中のみ必要な依存）** に追加します。

```bash
npm install -D package-name
## 省略形
npm i -D package-name
```

**使うべき場面**

> - 開発中にしか使わないツールやライブラリ
> - テストツール、Linter、ビルドツールなど

以下はアプリケーションが「動くため」には不要ですが、開発をスムーズにするための道具です。

```bash
npm i -D eslint
npm i -D typescript
npm i -D jest
```

## デプロイ時の流れ（Dockerfile）

依存を追加すると、`package.json` に以下のように追記されます。

本番環境では、`dependencies` のパッケージだけがインストールされます（`npm install --production` などを使った場合）。

```json
{
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "eslint": "^8.50.0"
  }
}
```

**🐳 Dockerfile の例：devDependencies を含めない本番用**

アプリケーションをデプロイするときはビルドステージと本番ステージに分けられています。それぞれインストールする依存パッケージが違います。

```docker:Dockerfile
## ===============================
## ビルドステージ
## ===============================

## 開発用の依存も含めてビルドするステージ（multi-stage build）
FROM node:20-alpine AS builder

## 作業ディレクトリ作成
WORKDIR /app

## package.json と package-lock.json を先にコピーして依存関係だけインストール
COPY package*.json ./

## devDependencies を含めて全部インストール
RUN npm ci

## ソースコードをコピー
COPY . .

## ビルド（例: TypeScript → JavaScript へ変換）
RUN npm run build


## ===============================
## 本番用ステージ（不要な依存を除去）
## ===============================

FROM node:20-alpine AS production

WORKDIR /app

## 必要なファイルだけコピー
COPY package*.json ./
## devDependencies を含めずにインストール（dependencies のみ）
RUN npm ci --omit=dev

## ビルド済みの成果物だけコピー
COPY --from=builder /app/dist ./dist

## アプリケーション起動
CMD ["node", "dist/index.js"]
```

**📁 ディレクトリ構成例**

```
.
├── Dockerfile
├── package.json
├── package-lock.json
├── tsconfig.json
├── src/
│   └── index.ts
└── dist/
    └── index.js （← ビルド後に生成）
```

**🔍 ステージの違い**

| ステージ   | 内容                                  | 備考                                 |
| ---------- | ------------------------------------- | ------------------------------------ |
| builder    | devDependencies を含めてビルド        | TypeScript, eslint, jest など利用 OK |
| production | `--omit=dev` で本番用だけインストール | セキュリティリスクや容量を削減       |

**💡 補足：`NODE_ENV=production`でも同様の効果あり**

```Dockerfile
ENV NODE_ENV=production
RUN npm install
```

> 似たような挙動になりますが、**`npm ci --omit=dev` の方が確実かつ再現性が高い**（CI/CD にも適している）ため、最近はそちらが推奨されることが多いです。

## おわりに

`npm i` と `npm i -D` は、単なる「依存を追加する」コマンドの違いではなく、**開発と本番の明確な分離**を実現するための大切な区分です。

自分のプロジェクトにとって「このライブラリは本番でも必要か？」と考えながら、正しく使い分けていきましょう。
