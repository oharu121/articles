# Box APIを使うためのOAuth2トークン管理設計

# はじめに

本記事では、**Box API を OAuth2 認証経由で利用する方法**と、**TypeScript でのトークン管理・API 呼び出しの実装例**を紹介します。

SDK を使わずに自前で実装したい場面（例：企業環境で自己署名証明書を使っている場合）に適した構成となっています。

>企業ネットワークなどで **自己署名証明書（Self-signed certificate）** を利用している場合、公式 SDK の利用は難しいため、**HTTP ベースでの実装**が必要です。

# 1. Box アプリの登録

まずは Box API を利用するために、アプリを作成・登録します。

**手順**

>1. [Box Developer Console](https://app.box.com/developers/console) にアクセス
>2. 「**Custom App (OAuth 2.0 with JWT or OAuth 2.0)**」を選択して新規作成
>3. **必要なスコープ（権限）を設定**
>→ 例：`Read and write all files and folders`
>4. （必要に応じて）**管理者の承認をリクエスト**
>→ 企業アカウントでは承認が必要なケースがあります

# 2. クライアント情報の取得と設定

以下の情報をメモしておきます。

>* `Client ID`
>* `Client Secret`
>* リダイレクト URI（Postman で確認用として以下を使用）：

  ```
  https://oauth.pstmn.io/v1/callback
  ```

# 3. 認可コードの取得

以下の URL にアクセスして、認可コードを取得します：

```
https://account.box.com/api/oauth2/authorize?client_id={Client ID}&response_type=code
```

認証が成功すると、以下のような URL にリダイレクトされます：

```
https://oauth.pstmn.io/v1/callback?state=&code=xxx
```

この `code` を控えておきましょう。

# 4. トークンの取得

以下の設定でアクセストークンとリフレッシュトークンを取得します。

**リクエスト設定**

* **Method**: POST
* **URL**: `https://api.box.com/oauth2/token`
* **Body**: `x-www-form-urlencoded`

| Key             | Value                                |
| --------------- | ------------------------------------ |
| `client_id`     | あなたの `Client ID`                     |
| `client_secret` | あなたの `Client Secret`                 |
| `code`          | 上記で取得した `code`                       |
| `redirect_uri`  | `https://oauth.pstmn.io/v1/callback` |
| `grant_type`    | `authorization_code`                 |

**レスポンス例**

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "expires_in": 3599,
  "token_type": "Bearer"
}
```

取得した `refresh_token` は、**安全な場所に保存してください**。

# 5. `Box` クラスの実装

**ストレージ設定例**

```json:storage.json
{
  "BOX_REFRESH_TOKEN": "xxx"
}
```

### 1. 認証状態の管理

```ts:Box.ts
class Box {
  private expiredAt = 0;
  private accessToken = "";

  private async checkToken(): Promise<void> {
    if (Date.now() >= this.expiredAt) {
      await this.refreshAccessToken();
    }
  }

export default new Box();
```

**📌 ポイント**

>* 毎回の API 実行前に `checkToken` を呼び出し、**トークンの期限切れを自動検出して更新**します。
>* `client_secret` や `refresh_token` は Git 管理対象外（`.gitignore`）とし、`.env` や Secrets Manager 等で管理しましょう
>* アクセストークンは **ログ出力や外部転送の対象にしない**よう徹底してください

### 2. refresh_token の読み書き

```ts:Box.ts
private getRefreshToken(): string {
  const path = "storage.json";
  const storage = JSON.parse(fs.readFileSync(path, "utf-8"));
  return storage.BOX_REFRESH_TOKEN;
}

private setRefreshToken(token: string): void {
  const path = "storage.json";
  const storage = JSON.parse(fs.readFileSync(path, "utf-8"));
  storage.BOX_REFRESH_TOKEN = token;
  fs.writeFileSync(path, JSON.stringify(storage, null, 2));
}
```

>Box の `refresh_token` は **毎回更新されるワンタイムトークン**です。使うたびに保存し直す必要があります。

### 3. トークンの更新処理

```ts:Box.ts
private async refreshAccessToken(): Promise<void> {
  const url = `https://api.box.com/oauth2/token`;
  const body = {
    client_id: credentials.BOX_CLIENT_ID,
    client_secret: credentials.BOX_CLIENT_SECRET,
    refresh_token: this.getRefreshToken(),
    grant_type: "refresh_token",
  };

  try {
    const res = await Postman.postWithEncodedString(url, body);
    if (res.status === 200) {
      this.accessToken = res.data.access_token;
      this.expiredAt = Date.now() + res.data.expires_in * 1000;
      this.setRefreshToken(res.data.refresh_token);
    } else {
      Logger.error(`Token refresh failed: ${res.status} ${JSON.stringify(res.data)}`);
      throw new Error("Token refresh failed");
    }
  } catch (error) {
    Logger.error(`Error refreshing token: ${error}`);
    throw error;
  }
}
```

### 4. ファイル情報の取得メソッド

```ts:Box.ts
public async getFileMeta(id: string): Promise<void> {
  await this.checkToken();

  const url = `https://api.box.com/2.0/files/${id}`;
  const res = await Postman.get(url, this.accessToken);
  console.log(JSON.stringify(res.data, null, 2));
}
```

* ファイル ID を指定して、Box API からファイルのメタ情報を取得します。

### 5. 使用例（スクリプト）

```ts:sandbox.ts
import "module-alias/register";
import Box from "@class/manager/Box";

(async () => {
  await Box.getFileMeta("1234567890"); // 対象ファイルIDを指定
  process.exit(0);
})();
```

**設計のポイントまとめ**

| 観点             | 内容                                        |
| -------------- | ----------------------------------------- |
| **責務の分離**      | 認証管理、トークン更新、API 実行をメソッド単位で整理              |
| **トークンの有効性検査** | `checkToken()` により自動リフレッシュ                |
| **SDK 非依存**    | 軽量かつ移植性の高い HTTP 実装                        |
| **安全性**        | リフレッシュトークンの管理はローカルファイルで分離し、`.env` との併用も可能 |

# おわりに

Box API は強力かつ柔軟ですが、**企業環境やセキュリティ要件によっては公式 SDK の利用が難しい場面**もあります。
本記事で紹介したように、**トークン管理やAPI通信をTypeScriptで自前実装すること**で、そうした制約を回避しつつ、安定した連携を実現できます。

また、今回の構成は以下のようなニーズにも対応できます：

>* トークンの自動更新による**安定運用**
>* SDKに依存しない**可搬性の高い実装**
>* セキュリティを考慮した**ローカル管理と分離設計**

今後は、**ファイルのアップロード・ダウンロード、フォルダ操作などの拡張例**も記事化していく予定です。
この記事が、Box API を安全かつ柔軟に利用したい方の参考になれば幸いです。