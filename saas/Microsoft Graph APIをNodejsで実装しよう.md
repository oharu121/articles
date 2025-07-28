# はじめに

本記事では **Microsoft Graph API** を利用するための手順を、以下の流れで解説します。

>- Azure ポータルでのアプリ登録
>- 認可コードとアクセストークンの取得（Postman使用）
>- `refresh_token` を使ったトークン更新
>- TypeScript でのトークン管理クラス実装例

Microsoft 365（Outlook、Teams、SharePoint など）と連携したい方は、ぜひ参考にしてください。

# セットアップ

以下の記事を参考しました。

https://qiita.com/Takuma_Kondo/items/3746c67b0b4f01f4e42f

### 1. Azureポータルでアプリ登録

1. Azure ポータルにアクセス→`https://portal.azure.com`
2. 左メニューから **「Microsoft Entra ID」** を選択  
3. 左メニューから **「App registrations」** → **「+ New registration」**

**登録時の入力項目**

| 項目 | 値 |
|------|----|
| **Name** | 任意のアプリ名 |
| **Supported account types** | *Accounts in this organizational directory only* |
| **Redirect URI** | Web を選択し、`https://oauth.pstmn.io/v1/callback` を入力 |

### 2. アプリ情報の確認

左メニューの **「Overview」**（または「General」）から、以下をメモします。

| 項目 | 内容 |
|------|------|
| **Application (client) ID** | クライアントID |
| **Directory (tenant) ID** | テナントID |

### 3. API Permissions の設定

1. 左メニューから **「API permissions」** を開く  
2. 「**Add a permission**」をクリック  
3. 「**Microsoft Graph**」→「**Delegated permissions**」を選択  
4. 必要な権限を追加  

**よく使う権限一覧**
| 必須 | 権限                         | 用途                                               |
|------|------------------------------|----------------------------------------------------|
| ✅   | `openid`                     | ユーザー識別子（sub）を取得するため               |
| ✅   | `offline_access`             | refresh token を取得して長期間認証を維持するため  |
|      | `ChannelMessage.Send`        | Teams のチャネルにメッセージを送信する             |
|      | `ChatMessage.Send`           | ユーザーとの 1:1 チャットにメッセージを送信する    |
|      | `Mail.Send`                  | Outlook（Microsoft 365）のメールを送信する          |
|      | `Mail.Read`                  | Outlook メールを読み取る                            |
|      | `Sites.ReadWrite.All`        | SharePoint のすべてのサイトに対する読み書き権限     |
|      | `Channel.ReadBasic.All`      | Teams チャネルの基本情報（名前など）を取得する     |
|      | `Calendars.ReadWrite`        | ユーザーのカレンダー予定の取得・追加・変更・削除   |
|      | `Calendars.ReadWrite.Shared` | 共有された他ユーザーのカレンダーに対して同様の操作 |

### 4. クライアントシークレットの作成

1. 左メニューから **「Certificates & secrets」** を選択  
2. 「**+ New client secret**」をクリック  
3. **Description** と **有効期限（最大24か月）** を設定  
4. 作成直後に表示される **Secret Value** を必ずメモ  

# Graph APIラッパークラス

### 1. クラスの実装

```typescript:Azure.ts
import credentials from "@utils/constants/crendentials";
const REDIRECT_URI = "https://soauth.pstmn.io/v1/callback";

const SCOPE = [
  "openid",
  "offline_access",
  "ChannelMessage.Send",
  "ChatMessage.Send",
  "Mail.Send",
  "Mail.Read",
  "User.Read",
  "Sites.ReadWrite.All",
  "Channel.ReadBasic.All",
  "Calendars.ReadWrite",
  "Calendars.ReadWrite.Shared",
];

class Azure {
  private expiredAt = Date.now();
  private refreshToken: string = "";
  private accessToken: string = "";

  private async checkToken(): Promise<void> {
    if (!this.refreshToken) {
      await this.forgeRefreshToken();
      return;
    }

    if (Date.now() >= this.expiredAt) {
      await this.refreshAccessToken();
    }
  }
}

export default new Graph();
```

### 2. 認可コードの取得

```typescript:Playwright.ts
public async getAzureCode(callback: string) {
  const browser = await chromium.launch();
  const context = await browser.newContext({ locale: "en-US" });
  const page = await context.newPage();

  try {
    await page.goto(callback);

    // Login process
    await page
      .getByLabel("Type your email address here")
      .fill(credentials.MAIL_ADDRESS);
    await page.getByRole("button", { name: "Next" }).click();
    await page
      .getByRole("link", { name: "Select to enter a code from" })
      .click();

    const otpInput = await page.getByLabel("Enter code from Okta Verify");
    const mfaOtp = Okta.generateOTP();

    await otpInput.fill(mfaOtp);
    await page.getByRole("button", { name: "Verify" }).click();
    await page.getByRole("button", { name: "No" }).click();
    await page
      .getByText("Your call is authenticated")
      .waitFor({ timeout: 60000 });

    const currentUrl = page.url();
    await browser.close();

    return currentUrl.split("code=")[1].split("&session_state")[0];
  } catch (error) {
    console.error("Error:", error);
  }
}
```

### 3. リフレッシュトークンの取得

```ts:Azure.ts
private async forgeRefreshToken() {
  const callback =
    `https://login.microsoftonline.com/${credentials.AZURE_TENANT_ID}/oauth2/v2.0/authorize?` +
    [
      `client_id=${credentials.AZURE_CLIENT_ID}`,
      "response_type=code",
      `redirect_uri=${REDIRECT_URI}`,
      `scope=${SCOPE.join("%20")}`,
      "response_mode=query",
    ].join("&");

  const code = await Playwright.getAzureCode(callback);
  const url = `https://login.microsoftonline.com/${credentials.AZURE_TENANT_ID}/oauth2/v2.0/token`;

  const reqTokenBody = {
    client_id: credentials.AZURE_CLIENT_ID,
    client_secret: credentials.AZURE_CLIENT_SECRET,
    code: code,
    redirect_uri: REDIRECT_URI,
    grant_type: "authorization_code",
    scope: SCOPE.join(" "),
  };

  try {
    const res = await Axon.new().encodeUrl().post(url, reqTokenBody);

    if (res.status === 200) {
      this.accessToken = res.data.access_token;
      this.expiredAt = Date.now() + res.data.expires_in * 1000;
      this.refreshToken = res.data.refresh_token;
    } else {
      Logger.error(
        `Failed to refresh access token: ${res.status} ${res.data}`
      );
      throw new Error("Failed to forge refresh token.");
    }
  } catch (error) {
    console.error("Error forging refresh token:", error);
    throw new Error("Error in forgeRefreshToken.");
  }
}
```

### 4. リフレッシュトークンの再発行

```ts:Azure.ts
private async refreshAccessToken(): Promise<void> {
  const url = `https://login.microsoftonline.com/${credentials.AZURE_TENANT_ID}/oauth2/v2.0/token`;

  const reqTokenBody = {
    client_id: credentials.AZURE_CLIENT_ID,
    scope: SCOPE.join(" "),
    refresh_token: this.refreshToken,
    redirect_uri: REDIRECT_URI,
    grant_type: "refresh_token",
    client_secret: credentials.AZURE_CLIENT_SECRET,
  };

  try {
    const res = await Axon.new().encodeUrl().post(url, reqTokenBody);

    if (res.status === 200) {
      this.accessToken = res.data.access_token;
      this.expiredAt = Date.now() + res.data.expires_in * 1000;
    } else {
      Logger.error(
        `Failed to refresh access token: ${res.status} ${res.data}`
      );
      throw new Error("Failed to refresh access token.");
    }
  } catch (error) {
    console.error("Error refreshing access token:", error);
    throw new Error("Error in refreshAccessToken.");
  }
}
```

## おわりに
以上で Microsoft Graph API を使うための準備と TypeScript 実装の一例まで解説しました。

Microsoft Graph API はとても強力で、**Teams の通知自動化**や**Outlook メール管理**、**SharePoint ファイル操作**など多彩な用途に活用できます。

必要に応じて、今回のクラスに `/me/messages` や `/teams/{team-id}/channels` などの追加実装も可能です。お気軽にコメントください！