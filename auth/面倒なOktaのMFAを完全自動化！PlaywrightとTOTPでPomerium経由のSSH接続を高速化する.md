# 面倒なOktaのMFAを完全自動化！PlaywrightとTOTPでPomerium経由のSSH接続を高速化する

# はじめに

「サーバーに接続するために、またOktaの認証か…」
「スマホを取り出して6桁のコードを入力するのが、地味に面倒…」

多くの開発現場で導入されているOktaなどのIdP（Identity Provider）は、セキュリティを高める一方で、日々の業務に小さな手間を積み重ねます。

この記事では、社内サーバーへのアクセスに利用していた **Pomerium + Okta認証** のフローを、**PlaywrightとTOTP（Time-based One-Time Password）** を使って完全自動化し、**10秒以内にSSH接続を確立する** 仕組みを構築した事例を紹介します。

 **😫 解決したかった課題：毎日の手動認証という「小さな苦痛」**

私たちのチームでは、Jump ServerへのアクセスにID認識型プロキシであるPomeriumを導入していました。接続時のフローは以下の通りです。

>1.  Pomerium Desktopを起動
>2.  ローカルでsshのコマンドを実行
>3.  ブラウザが自動で立ち上がり、Oktaのログインページにリダイレクトされる
>4.  ID/パスワードを入力してサインイン
>5.  スマートフォンでOkta Verifyを立ち上げ、ログインのリクエストを承認
>6.  認証が通り、ようやくTCPトンネルが確立される

このプロセス、特にステップ4が毎日の作業の中で無視できない手間となっており、チーム全体の生産性を少しずつ削っていました。

# 実装概要

**🚀 自動化のアーキテクチャ**

この手動フローを解消するため、以下の技術を組み合わせて認証ボットを構築しました。

>* **`Playwright`**: ブラウザ操作を自動化するライブラリ。Oktaのログイン画面を裏で操作します。
>* **`otplib`**: TOTPコードを生成するライブラリ。Okta Verifyのコードを自動で計算します。
>* **`Node.js (TypeScript)`**: 上記ライブラリを束ね、全体の処理を実行するランタイム。
>* **`child_process`**: PomeriumのクライアントバイナリをNode.jsから起動します。

**【自動化された処理フロー】**

>1.  Node.jsスクリプトがPomeriumプロセスを起動（`spawn`）
>2.  Pomeriumが出力する認証URLを標準エラー出力（`stderr`）から取得
>3.  Playwrightが取得したURLにバックグラウンドでアクセス
>4.  ID/パスワードを自動入力してログイン
>5.  `otplib` で生成したTOTPコードをMFA画面に自動入力
>6.  認証が完了し、SSHトンネルが確立！

# Pomeriumクラス：Pomeriumを起動し、認証URLをキャッチする

まず、`child_process.spawn` を使ってPomeriumのTCPトンネルコマンドを実行します。Pomeriumは認証が必要な場合、そのURLを標準エラー出力（`stderr`）に出力するため、これを監視してURLを正規表現で抜き出します。

### 1. クラスの構成

```ts:Pomerium.ts
import { spawn, ChildProcess } from "child_process";
import Playwright from "./Playwright"; // 後述

// --- 設定値 (外部ファイルや環境変数から読み込むのが望ましい) ---
const POMERIUM_CLI_PATH = "/opt/homebrew/bin/pomerium-cli";
const POMERIUM_TARGET_HOST = "ssh-jump-server.internal.corp";
const POMERIUM_LISTEN_PORT = "2222";

class Pomerium {
  private isConnected = false;
  private isLocked = false; // 「ロック」状態を管理するフラグ
  private pomeriumProcess: ChildProcess | null = null;
}

export default new Pomerium();
```

**📌 ポイント**

>* `isLocked` フラッグでの多重呼び出しを防ぐ
>* `isConnected` フラッグで接続が立っているかを判断

### 2. Pomeriumトンネルの開始・停止

- 開始

```ts:Pomerium.ts
public async start(): Promise<void> {
  if (this.isLocked || this.isConnected) {
    console.log("ℹ️ Process is locked or already connected.");
    return;
  }

  try {
    this.isLocked = true;
    console.log("🚀 Pomerium tunnel starting...");

    const authUrl = await this.getAuthUrl();
    await this.handleOktaLogin(authUrl);

    this.isConnected = true;
    console.log("✅ Pomerium tunnel established successfully.");
  } catch (error) {
    console.error("❌ Failed to establish Pomerium tunnel:", error);
    this.isConnected = false;
  } finally {
    this.isLocked = false;
  }
}
```

- 停止

```ts:Pomerium.ts
public stop(): void {
  if (this.pomeriumProcess) {
    console.log("🛑 Stopping Pomerium tunnel...");
    this.pomeriumProcess.kill();
    this.pomeriumProcess = null;
    this.isConnected = false;
    this.isLocked = false;
    console.log("✅ Pomerium tunnel stopped.");
  }
}
```

**📌 ポイント**

>* ロック中または接続済みなら、即座に処理を中断
>* 処理の開始前にロックを取得
>* 成功したら「接続済み」にする
>* 処理の完了後、必ずロックを解放
>* 停止時子プロセスを終了させる

### 3. 認証URLを取得する

```ts:Pomerium.ts
private getAuthUrl(): Promise<string> {
  return new Promise((resolve, reject) => {
    this.pomeriumProcess = spawn(POMERIUM_CLI_PATH, [
      "tcp",
      POMERIUM_TARGET_HOST,
      "--listen",
      `:${POMERIUM_LISTEN_PORT}`,
    ]);

    this.pomeriumProcess.stderr?.on("data", (chunk) => {
      const message = chunk.toString();
      const match = message.match(/https?:\/\/[^\s]+/g);
      if (match && match[0]) {
        resolve(match[0]);
      }
    });

    this.pomeriumProcess.on("error", (err) =>
      reject(new Error(`Failed to spawn Pomerium: ${err.message}`))
    );

    this.pomeriumProcess.on("close", (code) => {
      if (code !== 0 && !this.isConnected) {
        reject(new Error(`Pomerium process exited with code ${code}`));
      }
    });
  });
}
```

**📌 ポイント**

>* regexで認証URLを取得し、Playwrightに渡す

# Playwrightクラス：Okta認証を突破する

### 1. クラスの実装

```ts:Playright.ts
import { chromium } from "playwright";
import Okta from "./Okta"; // 後述

class Playwright {}
export default new Playwright();
```

### 2. 認証フローの実行

```ts:Playright.ts
public async handleOktaLogin(url: string): Promise<void> {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext({ locale: "en-US" });
  const page = await context.newPage();

  try {
    await page.goto(url);

    // IDとパスワード入力
    await page.getByLabel("Username").fill(credentials.OKTA_USERNAME);
    await page.getByLabel("Password").fill(credentials.OKTA_PASSWORD);
    await page.getByRole("button", { name: "Sign In" }).click();

    // MFAコード入力
    await page
      .getByLabel("Enter code")
      .waitFor({ state: "visible", timeout: 10000 });
    const otpCode = Okta.generateOtp();
    await page.getByLabel("Enter code").fill(otpCode);
    await page.getByRole("button", { name: "Verify" }).click();

    console.log("✅ Okta authentication successful!");
  } catch (error) {
    console.error("Playwright automation failed:", error);
    await page.screenshot({ path: "error_screenshot.png" });
    throw error;
  } finally {
    await browser.close();
  }
}
```

**📌 ポイント**

>* 自動化のためヘッドレスモードで実行
>* ロケールを指定することで、UIの言語を固定し、セレクタを安定させる
>* 失敗した場合、スクリーンショットを保存

# Oktaクラス：TOTPコードを生成してMFAをクリアする

### 1. クラスの実装

```ts:Okta.ts
import { totp } from "otplib";

class Okta {}
export default new Okta();
```

### 2. OTPの生成

```ts:Okta.ts
public generateOtp(): string {
  return totp.generate(credentials.OKTA_MFA_SECRET) || "";
}
```

> **🔑 TOTPシークレットの取得方法**
OktaでMFAを設定する際、通常はスマートフォンアプリでQRコードをスキャンします。このQRコードにはTOTPのシークレットキー（Base32形式の文字列）が含まれています。このキーを一度だけ手動で抽出し、環境変数として設定しておく必要があります。

https://qiita.com/oharu121/items/91e35ee7a31cfd80ef7e

 **⚠️ セキュリティに関する重要な注意点**

この仕組みは非常に便利ですが、**MFAの認証要素（ID, パスワード, TOTPシークレット）を環境変数として一箇所に集める**ため、取り扱いには細心の注意が必要です。

https://zenn.dev/oharu121/articles/2185c60debf88f

# 導入成果と今後の展望

>* **Pomeriumトンネル確立までの時間をほぼゼロに短縮**。コマンド一つで、10秒以内にSSH接続が可能な状態になりました。
>* 毎回のMFAコード入力の手間がなくなり、開発体験が大幅に向上しました。
>* 将来的には、この仕組みを応用して**GitHub ActionsなどのCI/CDパイプラインから、VPN経由でしかアクセスできない社内環境に対して自動でジョブを実行する**といった活用も検討しています。

PomeriumやOktaの認証が面倒だと感じている方は、この **Playwright＋TOTP** による自動化アプローチを試してみてはいかがでしょうか。