## Okta の QR から TOTP を抽出してスクリプトで OTP 自動生成する

## はじめに

Okta Verify による多要素認証（MFA）を使用していると、毎回スマホでコードを確認するのが面倒ですよね。

実は、Okta Verify の**QR コードから TOTP のシークレットキー**を取得し、**ローカル環境や Postman から OTP を生成**することが可能です。この記事では、**Postman で完結できる形**で、TOTP シークレットキーを取得する手順を解説します。

## 🧪 活用例

- Playwright などの E2E テストで自動ログインを実現
- CLI ツールから毎回スマホを使わず OTP 生成
- 自宅 PC から VPN ログイン用に OTP を使いたい場合 など

https://qiita.com/oharu121/items/b0101cd91b9f9e10bc5d

## 準備：QR コードの内容を確認

まず、Okta ポータルで以下の手順を行います。

1. Okta にログイン
2. 「セキュリティ方式」→「Okta Verify」
3. 「他のセットアップ」→「セットアップ」ボタンを押す
4. 表示された QR コードを読み取る（Okta アプリはダメ！）

QR コードの中身は、以下のような URL 形式です：

```
oktaverify://email@domain.com/?t=XXXXX&f=YYYYY&s=https
```

これから以下の値を取り出します：

- `XXXXX` → OTDT トークン
- `YYYYY` → Authenticator ID
- `DOMAIN` → Okta のドメイン（例: `yourcompany.okta.com`）

## 🔑 Step 1: 公開鍵の取得

まずは `clientInstanceKey.kid` と `n` を取得します。
Postman で以下の設定をしてください。

🔹 メソッド

```
GET https://DOMAIN.okta.com/oauth2/v1/keys
```

🔹 ヘッダー

なし（デフォルトで OK）

🔹 レスポンス例

```json
{
  "keys": [
    ...,
    {
      "alg": "RS256",
      "e": "AQAB",
      "kid": "OpSRC6wLx4oPnqGBUuLz-WL7_knbK_UhClzjvt1cpOw",
      "kty": "RSA",
      "n": "u0Y1ygDJ61AghDiEqeGW7lCv4iW2gLOON0Aw-Tm53xQ..."
    }
  ]
}
```

👆 一番最後のキーの `kid` と `n` を次で使います。

## 🔐 Step 2: sharedSecret の取得

次に、Postman で以下の POST リクエストを作成します。

🔹 メソッド

```
POST https://DOMAIN.okta.com/idp/authenticators
```

🔹 ヘッダー

```cmd
Accept: application/json; charset=UTF-8
Authorization: OTDT XXXXX
Content-Type: application/json; charset=UTF-8
User-Agent: D2DD7D3915.com.okta.android.auth/6.8.1 DeviceSDK/0.19.0 Android/7.1.1 unknown/Google
```

- `OTDT XXXXX` は QR コード中の `t` 値
- `Authorization` の前後にスペースが必要です

🔹 ボディ（JSON）

```json
{
  "authenticatorId": "YYYYY",
  "device": {
    "clientInstanceBundleId": "com.okta.android.auth",
    "clientInstanceDeviceSdkVersion": "DeviceSDK 0.19.0",
    "clientInstanceVersion": "6.8.1",
    "clientInstanceKey": {
      "alg": "RS256",
      "e": "AQAB",
      "okta:isFipsCompliant": false,
      "okta:kpr": "SOFTWARE",
      "kty": "RSA",
      "use": "sig",
      "kid": "先ほどの kid",
      "n": "先ほどの n"
    },
    "deviceAttestation": {},
    "displayName": "1Password",
    "fullDiskEncryption": false,
    "isHardwareProtectionEnabled": false,
    "manufacturer": "unknown",
    "model": "Google",
    "osVersion": "25",
    "platform": "ANDROID",
    "rootPrivileges": true,
    "screenLock": false,
    "secureHardwarePresent": false
  },
  "key": "okta_verify",
  "methods": [
    {
      "isFipsCompliant": false,
      "supportUserVerification": false,
      "type": "totp"
    }
  ]
}
```

🔹 レスポンス例

```json
{
  "methods": [
    {
      "sharedSecret": "JBSWY3DPEHPK3PXP"
    }
  ]
}
```

この `sharedSecret` が TOTP のシークレットキー（例）です。

## OTP 生成の確認

取得した `sharedSecret` を以下のように使えば、ローカルで OTP を生成できます。

```ts
import { TOTP } from "totp-generator";

const { otp } = TOTP.generate("JBSWY3DPEHPK3PXP");
console.log(otp);
```

## クラスで実装してみた

```typescript:Okta.ts
import Logger from "@class/common/Logger";
import Postman from "./Postman";
import axios from "axios";
import dayjs from "dayjs";
import { TOTP } from "totp-generator";
import credentials from "@utils/constants/credentials";

class Okta {
  public generateOTP() {
    const { otp, expires } = TOTP.generate(credentials.TOTP_SECRET_KEY);

    Logger.info(
      `will be expired at ${dayjs(expires).format("YYYY-MM-DD HH:mm:ss")}`
    );

    return otp;
  }

  public async getTotpSecret(domain: string, otdtToken: string, authenticatorId: string) {
    const url = `${domain}/idp/authenticators`;
    const { kid, n } = await this.getKeys(domain);

    const payload = {
      authenticatorId,
      device: {
        clientInstanceBundleId: "com.okta.android.auth",
        clientInstanceDeviceSdkVersion: "DeviceSDK 0.19.0",
        clientInstanceVersion: "6.8.1",
        clientInstanceKey: {
          alg: "RS256",
          e: "AQAB",
          "okta:isFipsCompliant": false,
          "okta:kpr": "SOFTWARE",
          kty: "RSA",
          use: "sig",
          kid,
          n,
        },
        displayName: "1Password",
        manufacturer: "unknown",
        model: "Google",
        osVersion: "25",
        platform: "ANDROID",
        rootPrivileges: true,
        screenLock: false,
        secureHardwarePresent: false,
      },
      key: "okta_verify",
      methods: [
        {
          isFipsCompliant: false,
          supportUserVerification: false,
          type: "totp",
        },
      ],
    };

    const headers = {
      Accept: "application/json;charset=UTF-8",
      Authorization: `OTDT ${otdtToken}`,
      "Content-Type": "application/json;charset=UTF-8",
      "User-Agent":
        "D2DD7D3915.com.okta.android.auth/6.8.1 DeviceSDK/0.19.0 Android/7.1.1 unknown/Google",
    };

    const res = await axios.post(url, payload, { headers });
    const method = res.data.methods.pop();
    return method.sharedSecret;
  }

  private async getKeys(domain: string) {
    const url = `${domain}/oauth2/v1/keys`;
    const res = await Postman.get(url);
    const key = res.data.keys.pop();
    const { kid, n } = key;
    return { kid, n };
  }
}

export default new Okta();
```

## 補足：ブラウザ拡張機能

下記を使うと便利です。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/f0bb17b0-999d-488a-8396-a084cd92939a.png)

https://chromewebstore.google.com/detail/authenticator/bhghoamapcdpbohphigoooaddinpkbai

## ⚠️ 注意点

- このリクエストを実行すると、Okta の `セキュリティ方式` に「1Password」という新しいデバイスが登録されます。
- QR コードから得られる情報は短時間で無効になります。失敗したら再発行してください。
- `clientInstanceKey` に関するエラー（400 など）が出た場合は、再度 `GET /oauth2/v1/keys` を実行して最新の鍵を取得しましょう。

## まとめ

Okta の QR コードと API を組み合わせれば、TOTP シークレットをローカルに保存していつでも OTP を生成可能です。セキュリティとのバランスを見ながら、適切に運用していきましょう。
| ステップ | 内容 | 備考 |
| ------ | -------------- | ------------------------------- |
| Step 1 | 公開鍵取得 | `/oauth2/v1/keys` を GET |
| Step 2 | sharedSecret 取得 | `/idp/authenticators` に POST |
| Step 3 | ローカルで OTP 生成 | `totp-generator`で利用可能 |

## 参考資料

下記の記事を参考しました。

https://gist.github.com/kamilhism/9f6f26ce3e10b6685af8c43f33aca808
