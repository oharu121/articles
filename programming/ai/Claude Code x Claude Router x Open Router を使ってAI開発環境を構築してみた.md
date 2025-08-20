## はじめに

[Claude Code](https://www.anthropic.com/claude-code) は、Anthropic が提供する AI アシスタント「Claude」を、開発者向けに特化させた CLI & GUI ツールです。

コード補完・設計レビュー・プロンプト改善などを **ローカル環境** で行え、開発支援を強化できます。

この記事では、以下をカバーします。

>* Claude Code のインストールと起動方法
>* Claude Router を経由して Claude Code を使う設定
>* 実際の活用例（Lyraによるプロンプト設計 → AI実装補助）
>* トラブルシューティング
>* 開発効率を高める操作コマンド

https://www.anthropic.com/claude-code

https://github.com/musistudio/claude-code-router

## Claude Code 単体での利用

### 1\. claude codeのインストール

```cmd
npm install -g @anthropic-ai/claude-code
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/de7bc7f9-af51-41bb-832f-6ae10386b26f.png)

### 2\. Claude Codeの起動

コマンドプロンプトに以下のコマンドを実行する

```cmd:コマンドプロンプト
claude
```

### 3\. 初期設定

>* **テーマ選択**（ライト / ダーク）
>* **アカウント連携**（Claude サブスクリプションまたは Anthropic Console アカウント）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/a47f7674-f610-4206-a0ab-465ba2b5c3e7.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/4cb6146d-a1af-4873-a130-a52caede1689.png)

## Claude Router 経由で Claude Code を使う

### 1\. Claude Routerのインストール

```cmd
npm install -g @musistudio/claude-code-router
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/31f46f4b-948d-4159-9694-01acf4d833ce.png)

### 2\. 設定ファイルの作成

>* macOS / Linux: `~/.claude-code-router/config.json`
>* Windows: `C:\Users\<ユーザー名>\config.json`

```json:config.json
{
  "Providers": [
    {
      "name": "openrouter",
      "api_base_url": "https://openrouter.ai/api/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": [
        "anthropic/claude-sonnet-4"
      ],
      "transformer": {
        "use": ["openrouter"]
      }
    }
  ],
  "Router": {
    "default": "openrouter,anthropic/claude-sonnet-4"
  },
  "LOG": true,
  "HOST": "127.0.0.1"
}
```

> Open Routerで生成されたAPI トークンを張り付ける。

https://openrouter.ai/

### 3\. Claude Routerの起動

`ccr code` を実行する

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/720af28f-0a65-4563-af4a-a6a5a1d47af1.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/54300e7f-794c-4c2c-a1cb-719871f7b0bf.png)

### 4\. トラブルシューティング

```
Service not running, starting service...
```

**💡 対処法**

`ccr start` を実行する

```
Error starting server: Error: listen EACCES: permission denied 127.0.0.1:3456
```

**💡 対処法**

設定ファイルに`PORT`を追加する

```diff_json:config.json
{
  "Providers": [
    {
      "name": "openrouter",
      "api_base_url": "https://openrouter.ai/api/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": [
        "anthropic/claude-sonnet-4"
      ],
      "transformer": {
        "use": ["openrouter"]
      }
    }
  ],
  "Router": {
    "default": "openrouter,anthropic/claude-sonnet-4"
  },
  "LOG": true,
  "HOST": "127.0.0.1",
+ "PORT": 23456
}
```

configを修正したら、`ccr strop`を実行して`ccr code`で変更を反映させる

## 実例：AI設計補助として使う

ここでは **Lyra プロンプト最適化**と組み合わせ、Claude Code でシステム設計を依頼する例を紹介します。

### 1\. Claude Router 起動 → プロジェクト開始

>* プロジェクトフォルダを作成する（`ai-guess-game`）
>* プロジェクトフォルダにコマンドプロンプトを開く
>* `ccr code` を実行する

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/87802990-0fcb-4e8f-87fe-9009110ee425.png)

### 2\. Lyraによる指示文の作成

>* 入力: 「Next.js でお絵描きゲームを作り、Gemini APIで絵の内容をAIが推測」

<details><summary>入力例</summary>

```
DETAIL using Claude —

## Project Overview

Create an online "Pictionary" game where players can draw on a canvas, and an AI system is responsible for guessing what the drawing represents.

## Technology Stack

* Framework: Next.js
* Use Gemini's API for AI, with API calls made from the backend
* Do not use any SDKs; call the Gemini API endpoint directly
```

</details>

>* Lyra が **最適化されたプロンプト**を生成し、Claude Code に渡す

<details><summary>実施結果</summary>

```
**Your Optimized Prompt:**

You're an expert AI product architect and full-stack developer.

I'm building an online "Pictionary"-style web app using **Next.js** where players draw on a canvas and an AI guesses what the drawing represents.

**Your mission:**
Help me design the core architecture and implementation flow for this app. I will use **Gemini's API directly from the backend (no SDKs)** to send user drawings to the AI.

Please provide:

1. **System Architecture**  
   - How to structure the frontend and backend in Next.js  
   - Where and how to process the drawing data for API calls  
   - How to securely call Gemini API directly without an SDK

2. **AI Integration Plan**  
   - How to convert the user's canvas drawing into an appropriate format (image or strokes)  
   - Best format to send to Gemini for accurate interpretation  
   - API request/response structure examples

3. **Gameplay Flow**  
   - Steps from player drawing to AI making a guess  
   - Whether to guess after each stroke or on demand (with pros/cons)  
   - Optional: Add top-N guess logic with confidence scores

4. **Security & Performance Tips**  
   - How to protect Gemini API key  
   - How to minimize latency for real-time interaction

Do not suggest using Gemini SDKs. All API calls should be handled manually.

Start by outlining the architecture, then walk through each step of the AI interaction and gameplay logic.
```

</details>

https://qiita.com/oharu121/items/cfe9d8b3850846a64d14

### 3\. Claude Code が設計図を提案

>* アーキテクチャ構成
>* API連携の流れ
>* ゲームプレイのステップ
>* セキュリティ・パフォーマンス対策

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/698d483e-afc5-45ae-ab32-a940738850c4.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/311fe0f9-b21b-4ab3-a1c2-e945bc2e1498.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/2eaaaabf-6588-4400-b9b1-e78052f071a2.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/619865b6-9b36-4143-b6eb-9d0150e75bfa.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/92dc20ff-1281-4ab8-8f4b-d1dd4468d800.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/5488a02d-8cab-4a2f-a8ae-faf474bc5499.png)

### 4\. コード生成と実行結果

>* 設計に沿って `implement with the above plan` を入力
>* 実装コードが生成され、描いた「豚」を正しく「豚の頭」と認識

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/45b80b2f-01d7-4725-b375-1209d28249e2.png)

>* 今回は4 USDをトップしましたが、0.87 USDを使いました

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/7a88f240-2857-4897-a225-106d00d79aba.png)

### 5\. 成果の確認

https://github.com/oharu121/ai-guess-game

>豚を描きましたが、「豚の頭」だと正しく認識してくれました！

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/9dcc311c-35f0-43da-826a-2c369b5b440f.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/21311a5e-0bc3-4f27-a7ff-48b8aab03132.png)

## Claude Code の活用法

Claude Code は、Chatだけでなく、以下のような「プロンプト設計支援機能」があります。

### 1\. Claude Code の便利コマンド

| コマンド       | 説明                          |
| ---------- | --------------------------- |
| `/init`    | プロジェクト情報を `CLAUDE.md` にまとめる |
| `/compact` | コンテキスト圧縮でトークン消費を削減          |
| `/clear`   | 会話履歴をリセット                   |
| `alt + m`  | モード切替（手動承認 / 自動承認 / プランモード） |

### 2\. VS Code 連携

>1\. VS Code に **Claude Code 拡張機能**をインストール

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/0b73eb07-1422-4c90-91c9-99406cf17ca6.png)

>2\. Claude Code で `/ide` を実行し、Visual Studio Code を選択

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/f5956b55-0bb9-4edd-9df2-865d5b50dd98.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/2a14765d-0386-4ec4-8351-9caaf16455cf.png)

>3\. 編集差分が可視化され、コードレビューの効率が向上

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/363d239c-7f24-4449-8085-a5305abb69e2.png)

## おわりに

これで、Claude Code をローカル環境で動かし、Claude Router と組み合わせて開発に活用する準備が整いました。次は実際の開発プロジェクトで、設計レビュー・コード生成・改善提案を試してみましょう。
