# Node.js × Cron で複数タスクを管理する方法

# はじめに

定期的にバックグラウンド処理を実行したい──バッチ処理、リマインダー、定期データ取得など、さまざまな場面で必要になります。

Node.jsでは、[`node-cron`](https://www.npmjs.com/package/node-cron) を使えば、cron形式のスケジュールに従って処理を自動実行できます。

この記事では、TypeScriptで **複数タスクを登録・停止・一括管理できるクラス** を構築し、実務でも使えるスケジューラを実装する方法を紹介します。

# `node-cron`とは？

`node-cron`は、UNIX系OSで広く使われているcronの構文を用いて、Node.jsで定期処理を簡単にスケジューリングできるライブラリです。

例えば、次のような記述が可能です：

```typescript
cron.schedule('0 * * * *', () => {
  console.log('毎時0分に実行されます');
});
```

**🗓️ Cron式の構成**

```plaintext
* * * * * *
│ │ │ │ │ │
│ │ │ │ │ └─ 曜日 (0 - 7)（0または7は日曜）
│ │ │ │ └── 月 (1 - 12)
│ │ │ └──── 日 (1 - 31)
│ │ └────── 時 (0 - 23)
│ └──────── 分 (0 - 59)
└───────── 秒（省略可）
```

例：`0 * * * *` → **毎時0分に実行**

# スケジューラクラスの実装

ここでは `TaskScheduler` クラスを実装し、以下のことを可能にします：

>* 複数の定期処理タスクを登録・管理
>* タスクの停止・再登録
>* 全タスクの一括停止

### 1. 型定義

まずは、タスク設定と内部管理用の型を定義します。

```ts
interface TaskConfig {
  name: string;
  cronExpression: string;
  command: () => Promise<void>;
}

interface ScheduledTasks {
  [taskName: string]: cron.ScheduledTask;
}
```

* `TaskConfig`：個別タスクの設定情報
* `ScheduledTasks`：実行中のタスクをマッピングして管理

### 2. `TaskScheduler` クラスの全体像

```ts:TaskScheduler.ts
class TaskScheduler {
  private scheduledTasks: ScheduledTasks = {};
  private taskConfig: TaskConfig[] = [
    {
      name: "task1",
      cronExpression: "0 * * * *",
      command: async () => {
        await Task1.main();
      }
    },
    // 必要に応じて他のタスクも追加
  ];
```

この `taskConfig` をもとにタスクを登録していきます。設定情報は外部JSONファイルやDBから読み込んでもOKです。

### 3. タスク登録関数 `scheduleTask`

```ts:TaskScheduler.ts
private scheduleTask(taskConfig: TaskConfig): void {
  const { name, cronExpression, command } = taskConfig;

  if (this.scheduledTasks[name]) {
    Logger.warning(`Task "${name}" already scheduled. Stopping existing task.`);
    this.scheduledTasks[name]!.stop();
  }

  const job = cron.schedule(cronExpression, async () => {
    try {
      await command();
    } catch (error) {
      Logger.error(`Error running ${name}: ${error}`);
    }
  });

  this.scheduledTasks[name] = job;
  Logger.info(`Task "${name}" scheduled with cron "${cronExpression}"`);
}
```

* すでに存在するタスクは一度停止（再スケジューリングに対応）
* 非同期関数（`async`）にも対応
* 実行ログやエラーを明確に記録

### 4. タスクの停止処理 `stopTask`

```ts:TaskScheduler.ts
public stopTask(taskName: string): void {
  if (this.scheduledTasks[taskName]) {
    this.scheduledTasks[taskName]!.stop();
    delete this.scheduledTasks[taskName];
    Logger.info(`Task "${taskName}" stopped.`);
  } else {
    Logger.warning(`Task "${taskName}" not found.`);
  }
}
```

指定タスクだけを安全に停止・削除できます。

### 5. 全タスクの停止 `stopAllTasks`

```ts:TaskScheduler.ts
public stopAllTasks(): void {
  Logger.task("Stopping all scheduled tasks...");
  for (const taskName in this.scheduledTasks) {
    this.scheduledTasks[taskName]!.stop();
    Logger.info(`Task "${taskName}" stopped.`);
  }
  this.scheduledTasks = {};
  Logger.success("All scheduled tasks stopped.");
}
```

全タスクを一括停止。特に終了処理やアプリのシャットダウン時に有用です。

# 実際の使い方

```ts
import TaskScheduler from './TaskScheduler';

const scheduler = new TaskScheduler();

// 起動時にすべてのタスクをスケジューリング
scheduler.startAllTasks?.(); // メソッドを定義してもOK
```

> `startAllTasks()` メソッドは、全ての `taskConfig` をループして `scheduleTask` を呼ぶ関数として別途定義してください。

# おわりに

この記事では、`node-cron` を使って **複数の定期処理を効率よく管理するクラスの設計方法** を紹介しました。

特に以下のポイントを重視しました：

>* cron式で柔軟にスケジュール設定
>* タスクの再スケジュールや停止処理の柔軟性
>* 実務でも拡張しやすい構成（DB連携、外部設定など）

定期バッチやリマインダー処理の実装に、ぜひご活用ください！