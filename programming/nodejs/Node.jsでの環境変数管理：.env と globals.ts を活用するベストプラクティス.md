## Node.js での環境変数管理：.env と globals.ts を活用するベストプラクティス

## はじめに

Node.js プロジェクトでは、**環境変数の管理がセキュリティと運用性に直結する重要な要素**です。
このブログでは、`.env` ファイルと `globals.ts` モジュールを組み合わせて、**安全で再利用しやすい設定管理の方法**を紹介します。

たとえば以下のようなケースで `.env` は有効です：

- DB 接続情報や API キーを **コードベースにハードコーディングしない**
- 開発・ステージング・本番など **環境ごとに設定を切り替えたい**
- セキュリティ面で **認証情報を Git に含めたくない**

## 1. `.env` による環境変数の定義

まずはプロジェクトのルートに `.env` ファイルを作成します。内容は以下のように、**キーと値のペア** で構成されます。

```bash:.env
## okta credentials
SSH_USERNAME="ssh-username"
SSH_PASSWORD="some-password"
## couchbase credentials
COUCHBASE_USERNAME="couchbase-user"
COUCHBASE_PASSWORD="couchbase-password"
## elastic DB
ELASTIC_AUTH="user:password"
## mysql
MYSOL_USERNAME="mysql-user"
MYSQL_PASSWORD="mysql-password"
```

**📌 ポイント**

> - `.env` は `.gitignore` に追加して、**バージョン管理には含めないようにしましょう**。

## 2. `globals.ts` で設定情報を一元管理

globals.ts ファイルは、アプリケーション内で共通して使用される設定情報を一元管理するためのモジュールです。このファイル内で、dotenv を使用して.env ファイルから環境変数を読み込み、必要な設定情報をエクスポートします。これにより、アプリケーション全体で一貫性のある設定情報を簡単に利用できるようになります。

まずは `dotenv` パッケージをインストール：

```
npm install dotenv --save-dev
```

次に、`globals.ts` を以下のように実装します：

```typescript:globals.ts
import os from "os";
import dotenv from "dotenv";

dotenv.config();

const sshUserName = process.env["SSH_USERNAME"];
const globals = {
  //jump server
  OS_USERNAME: os.userInfo().username,
  REMOTE_HOST: "<remote-ip>",
  KIBA_USERNAME: sshUserName,
  KIBA_PASSWORD: process.env["SSH_PASSWORD"] || "",
  LISTENING_PORT: 1234,
  CLUSTER_VIP: "<cluster-vip>",

  //robin
  NAMESPACE: "robin-namespace",
  TENANT: "robin-tenant",
  ROBIN_PORT: 12345,

  //couchbase
  COUCHBASE_HOST_DOMAIN: "host-domain(ipv4 or ipv6)",
  BUCKET: "bucket",
  COUCHBASE_USERNAME: process.env["COUCHBASE_USERNAME"],
  COUCHBASE_PASSWORD: process.env["COUCHBASE_PASSWORD"],

  //elastic database
  ELASTIC_AUTH: process.env["ELASTIC_AUTH"],

  //mysql database
  MYSQL_USERNAME: process.env["MYSQL_USERNAME"] || "",
  MYSQL_PASSWORD: process.env["MYSQL_PASSWORD"] || "",
  MYSQL_PORT: 1234,

  // project path
  REMOTE_DOC_PATH: `/home/${sshUserName}/documents`,
  LOCAL_DOC_PATH: "output/documents",
};

export default globals;
```

**📌 ポイント**

> - `.env` が未設定の場合でも動くよう、**`|| ""` でフォールバック**を指定
> - `os.userInfo().username` により、ローカルユーザー名も動的に取得

## 3. `globals.ts` を使って設定情報にアクセスする

設定値を使いたい他のファイル（例：`server.ts`）では、以下のように `globals` をインポートして利用します：

```typescript:server.ts
import express from 'express';
import globals from './globals';

const app = express();
const port = globals.port; // globals.ts からポート番号を取得

app.get('/', (req, res) => {
  res.send('Hello World!');
});

app.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```

## まとめ

`.env` + `globals.ts` による環境変数管理は、以下のメリットがあります：

- ✅ **コードと設定を明確に分離**できる
- ✅ **環境ごとに柔軟な切り替え**が可能
- ✅ **セキュリティ強化**（機密情報をリポジトリに含めない）
- ✅ **アプリ全体で一貫した設定の利用**が可能

本番運用やチーム開発においても、この構成は**保守性と安全性に優れたベストプラクティス**です。ぜひあなたのプロジェクトにも取り入れてみてください！
