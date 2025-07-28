# Node.js × Winstonで作るカスタムLoggerクラス

# はじめに

Node.jsアプリケーションの開発において、**ロギングは不可欠な要素**です。
適切なロギングは、

* アプリケーションの動作把握
* エラーの早期発見
* デバッグの効率化

につながります。

本記事では、Node.jsで広く使われているロギングライブラリ「[Winston](https://github.com/winstonjs/winston)」を使って、**カスタムLoggerクラス** を作成する手順を丁寧に解説します。

# Winstonとは？

Winstonは、以下の特徴を持つ**高機能なロギングライブラリ**です：

* **トランスポート機能**：ログの出力先（コンソール・ファイル・HTTPなど）を柔軟に設定
* **ログレベルの制御**：info / warn / error など、重要度に応じて出力制御
* **フォーマットのカスタマイズ**：JSONやタイムスタンプ付きログなど対応

多様な要件に対応できるため、本番環境にも適しています。

# 環境構築手順

### 1. Winstonのインストール

```bash
npm install winston
```

TypeScriptで開発する場合は、以下のようにセットアップしてください：

### 2. ディレクトリ構成

```plaintext
winston-example/
├── class/             # Loggerクラス本体
│   └── Logger.ts
├── constants/         # グローバル定数
│   └── globals.ts
├── logs/              # ログファイル格納場所
│   ├── combined.log
│   └── error.log
├── scripts/           # 実行スクリプト
│   └── example.ts
├── package.json
└── tsconfig.json
```

# Loggerクラスの実装

```ts:Logger.ts
import winston from "winston";
import globals from "../constants/globals";

class Logger {
  private logger: winston.Logger;

  constructor() {
    this.logger = winston.createLogger({
      level: "info",
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.File({
          filename: globals.ERROR_LOG_PATH,
          level: "error",
        }),
        new winston.transports.File({ filename: globals.COMBINED_LOG_PATH }),
      ],
    });
  }

  public info(msg: string) {
    if (!msg) return;
    msg.split("\n").filter(Boolean).forEach(line => {
      this.logger.info(`INFO: ${line}`);
      console.log("\x1b[34m%s \x1b[0m", `INFO: ${line}`);
    });
  }

  public warn(msg: string) {
    this.logger.warn(msg);
    console.log("\x1b[33m%s \x1b[0m", `WARNING: ${msg}`);
  }

  public error(msg: string) {
    this.logger.error(msg);
    console.log("\x1b[31m%s \x1b[0m", `ERROR: ${msg}`);
  }

  public task(msg: string) {
    this.logger.info(msg);
    console.log("\x1b[35m%s \x1b[0m", `TASK: ${msg}`);
  }

  public success(msg: string) {
    this.logger.info(msg);
    console.log("\x1b[32m%s \x1b[0m", `SUCCESS: ${msg}`);
  }
}

export default new Logger();
```

**📌 ポイント**

>* `info`, `warn`, `error`, `task`, `success` の5種類のログレベルを定義
>* コンソール出力にはANSIカラーを使用して視認性を向上
>* ファイルにはタイムスタンプ付きのJSON形式で保存

# Loggerの使い方

```ts
import Logger from "../class/Logger";

Logger.success("success");
Logger.info("info");
Logger.task("task");
Logger.warn("warn");
Logger.error("error");
```

**💻 ターミナル出力（例）**

![ターミナル出力](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/82d9d30c-e3ab-8ccb-c420-472acc9c23f5.png)

**`error.log` の出力例**

![error.log](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/e890a022-a2de-a9e8-0741-0c778ed00a1a.png)

**`combined.log` の出力例**

![combined.log](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/da0b968b-ba2a-1048-640e-171381f41aaa.png)

# おわりに

ロギングは開発・運用に欠かせない仕組みです。Winstonを使えば、**柔軟で拡張性の高いロガーを簡単に構築**できます。

今回紹介したLoggerクラスは、以下のような場面で特に役立ちます：

>* ローカル開発中のデバッグ
>* バッチ処理のログ記録
>* CI/CDパイプライン中の状態出力

ぜひ自分のプロジェクトにも取り入れて、**可視性の高いロギング体制** を整えてみてください！