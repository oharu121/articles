# PM2とTypeScriptで始めるプロセス管理：実装からデプロイまで完全ガイド

## はじめに

Node.jsアプリケーションの本番運用で「プロセス管理どうしよう...」と悩んだことはありませんか？

PM2（Process Manager 2）は、Node.jsアプリケーションの本番運用を強力にサポートするプロセスマネージャーです。今回は、TypeScriptプロジェクトでPM2を効果的に活用する方法を、基本設定から実装パターンまで詳しく解説します。

**🎯 この記事で学べること**

>- PM2の基本的な使い方と設定方法
>- TypeScriptプロジェクトでのPM2活用法
>- 実務で使える設定ファイルの書き方
>- モニタリングとログ管理のベストプラクティス
>- デプロイメント自動化の実装例

## PM2とは？

PM2は、Node.jsアプリケーション向けの高機能プロセスマネージャーです。

| 機能 | 説明 |
|------|------|
| プロセス管理 | アプリケーションの起動・停止・再起動を自動化 |
| クラスター化 | CPUコア数に応じてプロセスを複数起動 |
| 自動再起動 | クラッシュ時の自動復旧 |
| ログ管理 | ログの集約・ローテーション |
| モニタリング | リアルタイムでの監視機能 |

## インストールとセットアップ

### PM2のインストール

```bash
# グローバルインストール
npm install -g pm2

# または、プロジェクト内にインストール
npm install --save-dev pm2
```

### TypeScriptプロジェクトの基本構成

```
project/
├── src/
│   ├── app.ts
│   ├── server.ts
│   └── utils/
├── dist/
├── ecosystem.config.js
├── package.json
├── tsconfig.json
└── pm2.json
```

## TypeScript実装例

### 1\. Expressサーバー

```typescript:src/server.ts
import express from 'express';
import { createServer } from 'http';

export class Server {
  private app: express.Application;
  private server: any;
  private port: number;

  constructor(port: number = 3000) {
    this.app = express();
    this.port = port;
    this.setupMiddleware();
    this.setupRoutes();
  }

  private setupMiddleware(): void {
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
  }

  private setupRoutes(): void {
    this.app.get('/health', (req, res) => {
      res.json({
        status: 'OK',
        timestamp: new Date().toISOString(),
        pid: process.pid,
        uptime: process.uptime()
      });
    });

    this.app.get('/', (req, res) => {
      res.json({
        message: 'Hello from PM2 + TypeScript!',
        pid: process.pid,
        env: process.env.NODE_ENV || 'development'
      });
    });
  }

  public start(): Promise<void> {
    return new Promise((resolve) => {
      this.server = this.app.listen(this.port, () => {
        console.log(`🚀 Server running on port ${this.port} (PID: ${process.pid})`);
        resolve();
      });
    });
  }

  public stop(): Promise<void> {
    return new Promise((resolve) => {
      if (this.server) {
        this.server.close(() => {
          console.log('🛑 Server stopped gracefully');
          resolve();
        });
      } else {
        resolve();
      }
    });
  }
}
```

### 2\. エントリーポイント

```typescript:src/app.ts
import { Server } from './server';

class Application {
  private server: Server;

  constructor() {
    this.server = new Server(Number(process.env.PORT) || 3000);
    this.setupProcessHandlers();
  }

  private setupProcessHandlers(): void {
    // グレースフルシャットダウンの実装
    process.on('SIGTERM', async () => {
      console.log('📥 SIGTERM received, shutting down gracefully...');
      await this.shutdown();
    });

    process.on('SIGINT', async () => {
      console.log('📥 SIGINT received, shutting down gracefully...');
      await this.shutdown();
    });

    // PM2からのシャットダウンシグナル
    process.on('SIGKILL', async () => {
      console.log('📥 SIGKILL received, shutting down...');
      await this.shutdown();
    });

    // 未処理の例外をキャッチ
    process.on('uncaughtException', (error) => {
      console.error('💥 Uncaught Exception:', error);
      this.shutdown();
    });

    process.on('unhandledRejection', (reason) => {
      console.error('💥 Unhandled Rejection:', reason);
      this.shutdown();
    });
  }

  private async shutdown(): Promise<void> {
    try {
      await this.server.stop();
      process.exit(0);
    } catch (error) {
      console.error('❌ Error during shutdown:', error);
      process.exit(1);
    }
  }

  public async start(): Promise<void> {
    try {
      await this.server.start();
      console.log('✅ Application started successfully');
    } catch (error) {
      console.error('❌ Failed to start application:', error);
      process.exit(1);
    }
  }
}

// アプリケーション起動
const app = new Application();
app.start();
```

### 3\. ecosystem.config.js（推奨）

```javascript:ecosystem.config.js
// TypeScript開発用

module.exports = {
  apps: [
    {
      name: 'my-app-dev',
      script: 'ts-node',
      args: 'src/app.ts',
      interpreter: 'node',
      watch: ['src'],
      ignore_watch: ['node_modules', 'dist'],
      env: {
        NODE_ENV: 'development',
        TS_NODE_PROJECT: './tsconfig.json'
      }
    },
    {
      name: 'my-app-prod',
      script: 'dist/app.js',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production'
      },
      // 自動再起動設定
      watch: false,
      ignore_watch: ['node_modules', 'logs'],
      max_restarts: 10,
      min_uptime: '10s',
      max_memory_restart: '1G',
      
      // ログ設定
      log_file: './logs/combined.log',
      out_file: './logs/out.log',
      error_file: './logs/error.log',
      time: true,
      
      // その他の設定
      kill_timeout: 5000,
      wait_ready: true,
      listen_timeout: 3000
    }
  ],
  deploy: {
    production: {
      user: 'deploy',
      host: ['your-server.com'],
      ref: 'origin/main',
      repo: 'git@github.com:username/repo.git',
      path: '/var/www/production',
      'pre-deploy-local': '',
      'post-deploy': 'npm install && npm run build && pm2 reload ecosystem.config.js --env production',
      'pre-setup': ''
    }
  }
};
```

### 4\. package.jsonスクリプト

```json
{
  "name": "pm2-typescript-example",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "dev": "ts-node src/app.ts",
    "start": "pm2-runtime start ecosystem.config.js",
    "pm2:start": "pm2 start ecosystem.config.js",
    "pm2:stop": "pm2 stop ecosystem.config.js",
    "pm2:restart": "pm2 restart ecosystem.config.js",
    "pm2:reload": "pm2 reload ecosystem.config.js",
    "pm2:delete": "pm2 delete ecosystem.config.js",
    "pm2:logs": "pm2 logs",
    "pm2:monit": "pm2 monit",
    "pm2:dev": "pm2 start ecosystem.config.js --env development",
    "pm2:prod": "npm run build && pm2 start ecosystem.config.js --env production"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/node": "^18.0.0",
    "ts-node": "^10.0.0",
    "typescript": "^4.8.0"
  }
}
```

## 実務でよく使うPM2コマンド

### 1\. 基本操作

```bash
# アプリケーション起動
pm2 start ecosystem.config.js

# 環境指定での起動
pm2 start ecosystem.config.js --env production

# アプリケーション一覧表示
pm2 list

# ログ表示
pm2 logs my-app

# リアルタイム監視
pm2 monit

# アプリケーション停止
pm2 stop my-app

# アプリケーション削除
pm2 delete my-app

# 全アプリケーション停止
pm2 stop all

# 設定リロード（ダウンタイムなし）
pm2 reload ecosystem.config.js
```

### 2\. ログ管理

```bash
# ログローテーション設定
pm2 install pm2-logrotate

# ログファイルの場所確認
pm2 logs --lines 100

# ログクリア
pm2 flush

# 特定アプリのログのみ表示
pm2 logs my-app --lines 50
```

## モニタリングとヘルスチェック

### 1\. カスタムヘルスチェックの実装

```typescript
// src/utils/healthCheck.ts
export interface HealthStatus {
  status: 'healthy' | 'unhealthy';
  timestamp: string;
  uptime: number;
  memory: NodeJS.MemoryUsage;
  pid: number;
  version: string;
}

export class HealthChecker {
  public static getStatus(): HealthStatus {
    return {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      memory: process.memoryUsage(),
      pid: process.pid,
      version: process.version
    };
  }

  public static async performDeepCheck(): Promise<HealthStatus & { checks: any }> {
    const baseStatus = this.getStatus();
    
    const checks = {
      database: await this.checkDatabase(),
      redis: await this.checkRedis(),
      disk: await this.checkDiskSpace(),
      memory: this.checkMemoryUsage()
    };

    const isHealthy = Object.values(checks).every(check => check.status === 'ok');
    
    return {
      ...baseStatus,
      status: isHealthy ? 'healthy' : 'unhealthy',
      checks
    };
  }

  private static async checkDatabase(): Promise<{ status: string; latency?: number }> {
    // データベース接続チェックの実装
    return { status: 'ok', latency: 10 };
  }

  private static async checkRedis(): Promise<{ status: string; latency?: number }> {
    // Redis接続チェックの実装
    return { status: 'ok', latency: 5 };
  }

  private static async checkDiskSpace(): Promise<{ status: string; free: string }> {
    // ディスク容量チェックの実装
    return { status: 'ok', free: '80%' };
  }

  private static checkMemoryUsage(): { status: string; usage: string } {
    const usage = process.memoryUsage();
    const usedMB = Math.round(usage.heapUsed / 1024 / 1024);
    
    return {
      status: usedMB < 512 ? 'ok' : 'warning',
      usage: `${usedMB}MB`
    };
  }
}
```

### 2\. PM2モニタリング連携

```typescript
// src/utils/pm2Monitor.ts
import * as pm2 from 'pm2';

export class PM2Monitor {
  public static async getProcessInfo(appName: string): Promise<any> {
    return new Promise((resolve, reject) => {
      pm2.describe(appName, (err, processDescription) => {
        if (err) {
          reject(err);
        } else {
          resolve(processDescription);
        }
      });
    });
  }

  public static async restartApp(appName: string): Promise<void> {
    return new Promise((resolve, reject) => {
      pm2.restart(appName, (err) => {
        if (err) {
          reject(err);
        } else {
          resolve();
        }
      });
    });
  }

  public static async reloadApp(appName: string): Promise<void> {
    return new Promise((resolve, reject) => {
      pm2.reload(appName, (err) => {
        if (err) {
          reject(err);
        } else {
          resolve();
        }
      });
    });
  }
}
```

## 実務でのベストプラクティス

### 1\. グレースフルシャットダウンの実装

```typescript
// 接続中のリクエストを適切に処理してからシャットダウン
private async gracefulShutdown(): Promise<void> {
  console.log('Starting graceful shutdown...');
  
  // 新しいリクエストの受付を停止
  this.server.close(() => {
    console.log('HTTP server closed');
  });
  
  // 既存の接続が完了するまで待機（最大30秒）
  const timeout = setTimeout(() => {
    console.log('Forcefully shutting down');
    process.exit(1);
  }, 30000);
  
  // 全ての接続が閉じられたらタイムアウトをクリア
  this.server.on('close', () => {
    clearTimeout(timeout);
    process.exit(0);
  });
}
```

### 2\. メモリリーク対策

```typescript
// 定期的なメモリ使用量チェック
setInterval(() => {
  const usage = process.memoryUsage();
  const usedMB = Math.round(usage.heapUsed / 1024 / 1024);
  
  if (usedMB > 512) { // 512MB超過時
    console.warn(`High memory usage: ${usedMB}MB`);
    // 必要に応じてプロセス再起動
  }
}, 60000); // 1分間隔
```

### 3\. 環境別設定の分離

```typescript
// src/config/index.ts
interface Config {
  port: number;
  nodeEnv: string;
  logLevel: string;
  db: {
    host: string;
    port: number;
  };
}

export const config: Config = {
  port: Number(process.env.PORT) || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
  logLevel: process.env.LOG_LEVEL || 'info',
  db: {
    host: process.env.DB_HOST || 'localhost',
    port: Number(process.env.DB_PORT) || 5432
  }
};
```

###\/ 4\. Dockerとの組み合わせ

```dockerfile:dockerfile
FROM node:18-alpine

WORKDIR /app

# PM2をグローバルインストール
RUN npm install -g pm2

# 依存関係のインストール
COPY package*.json ./
RUN npm ci --only=production

# TypeScriptビルド
COPY . .
RUN npm run build

# 本番環境でのみ必要なファイルを残す
RUN rm -rf src tsconfig.json

EXPOSE 3000

# PM2でアプリケーション起動
CMD ["pm2-runtime", "start", "ecosystem.config.js", "--env", "production"]
```

```yaml:docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    
  # PM2の監視用サービス（オプション）
  pm2-monitor:
    build: .
    command: pm2 web
    ports:
      - "9615:9615"
    depends_on:
      - app
```

### 5\. CI/CDパイプライン例

```yaml:.github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Build TypeScript
      run: npm run build
    
    - name: Run tests
      run: npm test
    
    - name: Deploy to server
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.KEY }}
        script: |
          cd /var/www/production
          git pull origin main
          npm ci --only=production
          npm run build
          pm2 reload ecosystem.config.js --env production
```

## まとめ

PM2とTypeScriptを組み合わせることで、以下のメリットが得られます：

**✅ 得られる効果**

>- **高可用性**: クラスター化による障害耐性
>- **簡単な運用**: 設定ファイルによる一元管理
>- **モニタリング**: リアルタイムでの監視・ログ管理
>- **開発効率**: TypeScriptの型安全性と開発体験

**🚀 次のステップ**

>1\. **実際にプロジェクトで試してみる**
>2\. **モニタリングツールとの連携**（New Relic、DataDogなど）
>3\. **Kubernetesでの運用パターン**を学ぶ
>4\. **マイクロサービス**でのPM2活用

PM2を使うことで、Node.js + TypeScriptアプリケーションの本番運用が格段に楽になります。ぜひ今回の実装例を参考に、あなたのプロジェクトでも試してみてください！