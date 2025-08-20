# もう認証で迷わない！Lucia + TypeScriptで作るセキュアなWebアプリ認証システム完全ガイド

## はじめに

Webアプリの認証機能を実装する際、こんな悩みはありませんか？

>- **認証ライブラリが複雑すぎて理解に時間がかかる**
>- **セキュリティホールを作らない自信がない**  
>- **TypeScriptで型安全な認証を実現したい**
>- **セッション管理やパスワードハッシュ化を安全に行いたい**

従来の認証ライブラリは機能が豊富な分、学習コストが高く、初心者には扱いづらいものが多いのが現実です。

そこで今回紹介するのが **Lucia** です！Luciaは「シンプルで安全」をコンセプトにしたTypeScript製の認証ライブラリで、最小限の設定で本格的な認証システムを構築できます。

この記事では、Luciaの特徴から実際のTypeScriptでの実装例まで、実務で使える形で詳しく解説していきますね。

## Luciaとは？なぜ選ぶべきなのか

**📕 Luciaの基本概念**

Luciaは2023年に登場した比較的新しい認証ライブラリで、以下の特徴があります：

| 特徴 | 詳細 |
|------|------|
| **TypeScript ファースト** | 型安全性を重視した設計 |
| **ミニマリスト設計** | 必要な機能のみに絞った軽量ライブラリ |
| **フレームワーク非依存** | Node.js、Deno、Cloudflare Workersで動作 |
| **セッション管理** | 安全なセッション管理機能を内蔵 |
| **パスワードハッシュ化** | Argon2による強力なハッシュ化 |

**🎯 他のライブラリとの比較**

| ライブラリ | 学習コスト | TypeScript対応 | 設定の複雑さ | セッション管理 |
|-----------|-----------|--------------|-------------|-------------|
| **Lucia** | ⭐⭐ | ✅ 完全対応 | ⭐⭐ | ✅ 内蔵 |
| Passport | ⭐⭐⭐⭐ | 🔶 部分的 | ⭐⭐⭐⭐ | ❌ 別途必要 |
| Auth0 | ⭐⭐⭐ | ✅ 対応 | ⭐⭐⭐ | ✅ 提供 |
| Firebase Auth | ⭐⭐ | ✅ 対応 | ⭐⭐ | ✅ 提供 |

**💡 Luciaが適している場面**

>* TypeScriptで型安全な認証を実装したい
>* シンプルな認証機能で十分（メール/パスワード、セッション管理）
>* 学習コストを抑えてすぐに使いたい
>* 自前でカスタマイズしやすい認証システムが欲しい

## 基本セットアップ

### 1\. 必要なパッケージのインストール

```typescript:package.json
{
  "dependencies": {
    "lucia": "^3.0.0",
    "@lucia-auth/adapter-prisma": "^4.0.0",
    "prisma": "^5.0.0",
    "@prisma/client": "^5.0.0",
    "argon2": "^0.31.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

### 2\. データベーススキーマの設定

```typescript:prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = "file:./dev.db"
}

model User {
  id       String @id @unique
  username String @unique
  email    String @unique
  hashedPassword String
  
  sessions Session[]
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("users")
}

model Session {
  id        String   @id
  userId    String
  expiresAt DateTime
  
  user User @relation(references: [id], fields: [userId], onDelete: Cascade)

  @@map("sessions")
}
```

### 3\. Luciaの初期設定

```typescript:src/lib/auth.ts
import { Lucia } from "lucia";
import { PrismaAdapter } from "@lucia-auth/adapter-prisma";
import { PrismaClient } from "@prisma/client";

const client = new PrismaClient();

const adapter = new PrismaAdapter(client.session, client.user);

export const lucia = new Lucia(adapter, {
  sessionCookie: {
    expires: false,
    attributes: {
      secure: process.env.NODE_ENV === "production"
    }
  },
  getUserAttributes: (attributes) => {
    return {
      username: attributes.username,
      email: attributes.email
    };
  }
});

declare module "lucia" {
  interface Register {
    Lucia: typeof lucia;
    DatabaseUserAttributes: {
      username: string;
      email: string;
    };
  }
}
```

**📌 設定のポイント**

>* `sessionCookie.expires: false` で永続的なセッションクッキーを設定
>* `secure: true` で本番環境でのHTTPS必須化
>* `getUserAttributes` でセッションから取得できるユーザー情報を定義

## 実践：ユーザー登録とログイン機能の実装

### 1\. ユーザー登録機能

```typescript:src/controllers/auth.ts
import { hash } from "argon2";
import { generateId } from "lucia";
import { lucia } from "../lib/auth.js";
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

export async function signUp(
  username: string, 
  email: string, 
  password: string
) {
  try {
    // パスワードのハッシュ化
    const hashedPassword = await hash(password);
    
    // 一意のユーザーIDを生成
    const userId = generateId(15);
    
    // ユーザーをデータベースに保存
    const user = await prisma.user.create({
      data: {
        id: userId,
        username,
        email,
        hashedPassword
      }
    });

    // セッションを作成
    const session = await lucia.createSession(userId, {});
    const sessionCookie = lucia.createSessionCookie(session.id);

    return {
      user: {
        id: user.id,
        username: user.username,
        email: user.email
      },
      sessionCookie
    };
  } catch (error) {
    if (error instanceof Error && error.message.includes('Unique constraint')) {
      throw new Error("ユーザー名またはメールアドレスが既に使用されています");
    }
    throw new Error("ユーザー登録に失敗しました");
  }
}
```

### 2\. ログイン機能

```typescript:src/controllers/auth.ts
import { verify } from "argon2";

export async function signIn(username: string, password: string) {
  try {
    // ユーザーの検索
    const existingUser = await prisma.user.findUnique({
      where: { username }
    });

    if (!existingUser) {
      throw new Error("ユーザー名またはパスワードが正しくありません");
    }

    // パスワードの検証
    const validPassword = await verify(existingUser.hashedPassword, password);
    
    if (!validPassword) {
      throw new Error("ユーザー名またはパスワードが正しくありません");
    }

    // セッションの作成
    const session = await lucia.createSession(existingUser.id, {});
    const sessionCookie = lucia.createSessionCookie(session.id);

    return {
      user: {
        id: existingUser.id,
        username: existingUser.username,
        email: existingUser.email
      },
      sessionCookie
    };
  } catch (error) {
    throw new Error("ログインに失敗しました");
  }
}
```

### 3\. ログアウト機能

```typescript:src/controllers/auth.ts
export async function signOut(sessionId: string) {
  try {
    await lucia.invalidateSession(sessionId);
    const sessionCookie = lucia.createBlankSessionCookie();
    
    return { sessionCookie };
  } catch (error) {
    throw new Error("ログアウトに失敗しました");
  }
}
```

## Express.jsとの統合例

### 1\. ミドルウェアの作成

```typescript:src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import { lucia } from '../lib/auth.js';
import { verifyRequestOrigin } from 'lucia';

declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        username: string;
        email: string;
      };
      session?: {
        id: string;
        userId: string;
        expiresAt: Date;
      };
    }
  }
}

export async function authMiddleware(
  req: Request, 
  res: Response, 
  next: NextFunction
) {
  // CSRF保護（POST、PUT、DELETEリクエストの場合）
  if (req.method !== "GET") {
    const originHeader = req.headers.origin;
    const hostHeader = req.headers.host;
    
    if (!originHeader || !hostHeader || 
        !verifyRequestOrigin(originHeader, [hostHeader])) {
      return res.status(403).json({ error: 'Forbidden' });
    }
  }

  const sessionId = req.cookies.auth_session ?? null;
  
  if (!sessionId) {
    req.user = null;
    req.session = null;
    return next();
  }

  const { session, user } = await lucia.validateSession(sessionId);
  
  if (session && session.fresh) {
    const sessionCookie = lucia.createSessionCookie(session.id);
    res.cookie(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
  }
  
  if (!session) {
    const sessionCookie = lucia.createBlankSessionCookie();
    res.cookie(sessionCookie.name, sessionCookie.value, sessionCookie.attributes);
  }

  req.session = session;
  req.user = user;
  next();
}
```

### 2\. APIルートの実装

```typescript:src/routes/auth.ts
import express from 'express';
import { signUp, signIn, signOut } from '../controllers/auth.js';
import { authMiddleware } from '../middleware/auth.js';

const router = express.Router();

// ユーザー登録
router.post('/signup', authMiddleware, async (req, res) => {
  try {
    const { username, email, password } = req.body;
    
    // バリデーション
    if (!username || !email || !password) {
      return res.status(400).json({ 
        error: 'ユーザー名、メールアドレス、パスワードは必須です' 
      });
    }

    if (password.length < 6) {
      return res.status(400).json({ 
        error: 'パスワードは6文字以上である必要があります' 
      });
    }

    const result = await signUp(username, email, password);
    
    res.cookie(
      result.sessionCookie.name, 
      result.sessionCookie.value, 
      result.sessionCookie.attributes
    );
    
    res.status(201).json({ 
      message: 'ユーザー登録が完了しました',
      user: result.user 
    });
  } catch (error) {
    res.status(400).json({ 
      error: error instanceof Error ? error.message : '登録に失敗しました' 
    });
  }
});

// ログイン
router.post('/signin', authMiddleware, async (req, res) => {
  try {
    const { username, password } = req.body;
    
    if (!username || !password) {
      return res.status(400).json({ 
        error: 'ユーザー名とパスワードは必須です' 
      });
    }

    const result = await signIn(username, password);
    
    res.cookie(
      result.sessionCookie.name, 
      result.sessionCookie.value, 
      result.sessionCookie.attributes
    );
    
    res.json({ 
      message: 'ログインしました',
      user: result.user 
    });
  } catch (error) {
    res.status(401).json({ 
      error: error instanceof Error ? error.message : 'ログインに失敗しました' 
    });
  }
});

// ログアウト
router.post('/signout', authMiddleware, async (req, res) => {
  try {
    if (!req.session) {
      return res.status(401).json({ error: 'ログインしていません' });
    }

    const result = await signOut(req.session.id);
    
    res.cookie(
      result.sessionCookie.name, 
      result.sessionCookie.value, 
      result.sessionCookie.attributes
    );
    
    res.json({ message: 'ログアウトしました' });
  } catch (error) {
    res.status(500).json({ 
      error: error instanceof Error ? error.message : 'ログアウトに失敗しました' 
    });
  }
});

// 認証が必要なルートの例
router.get('/me', authMiddleware, (req, res) => {
  if (!req.user) {
    return res.status(401).json({ error: 'ログインが必要です' });
  }

  res.json({ user: req.user });
});

export default router;
```

## セキュリティ対策と最適化

### 1\. セキュリティベストプラクティス

```typescript:src/lib/security.ts
// レート制限の実装例
import rateLimit from 'express-rate-limit';

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 5, // 最大5回のログイン試行
  message: {
    error: 'ログイン試行回数が上限に達しました。15分後に再試行してください。'
  },
  standardHeaders: true,
  legacyHeaders: false,
});

// パスワード強度チェック
export function validatePassword(password: string): string[] {
  const errors: string[] = [];
  
  if (password.length < 8) {
    errors.push('パスワードは8文字以上である必要があります');
  }
  
  if (!/[A-Z]/.test(password)) {
    errors.push('大文字を含める必要があります');
  }
  
  if (!/[a-z]/.test(password)) {
    errors.push('小文字を含める必要があります');
  }
  
  if (!/\d/.test(password)) {
    errors.push('数字を含める必要があります');
  }
  
  return errors;
}

// メールアドレス検証
export function validateEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}
```

### 2\. パフォーマンス最適化

```typescript:src/lib/cache.ts
// セッションキャッシュの実装（Redis使用例）
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

export async function getCachedSession(sessionId: string) {
  try {
    const cached = await redis.get(`session:${sessionId}`);
    return cached ? JSON.parse(cached) : null;
  } catch (error) {
    console.error('Cache get error:', error);
    return null;
  }
}

export async function setCachedSession(sessionId: string, session: any, ttl = 3600) {
  try {
    await redis.setex(`session:${sessionId}`, ttl, JSON.stringify(session));
  } catch (error) {
    console.error('Cache set error:', error);
  }
}

export async function deleteCachedSession(sessionId: string) {
  try {
    await redis.del(`session:${sessionId}`);
  } catch (error) {
    console.error('Cache delete error:', error);
  }
}
```

## トラブルシューティング

**🪄 よくある問題と解決策**

| 問題 | 原因 | 解決策 |
|------|------|--------|
| **セッションが保存されない** | クッキー設定の問題 | `secure: false` を開発環境で設定 |
| **型エラーが発生** | declare module未設定 | `lucia` モジュールの型宣言を追加 |
| **ログイン後にリダイレクトしない** | セッション検証の失敗 | ミドルウェアでセッション状態を確認 |
| **CSRF エラー** | Origin ヘッダーの不一致 | `verifyRequestOrigin` の設定を見直し |

**👍 デバッグ用のヘルパー関数**

```typescript:src/utils/debug.ts
export function debugSession(req: Express.Request) {
  console.log('=== セッション情報 ===');
  console.log('Cookie:', req.cookies.auth_session);
  console.log('Session:', req.session);
  console.log('User:', req.user);
  console.log('==================');
}

export function logAuthEvent(event: string, userId?: string, details?: any) {
  console.log(`[AUTH] ${new Date().toISOString()} - ${event}`, {
    userId,
    ...details
  });
}
```

## まとめ

Luciaを使用することで、TypeScriptでの認証システム実装が格段に簡単になります！

✅ **この記事で学んだこと**

>* Luciaの基本概念と他ライブラリとの違い
>* TypeScriptでの型安全な実装方法
>* Express.jsとの統合パターン
>* セキュリティ対策の実装方法
>* よくある問題の解決策

🎯 **次にやってみてください**

>1\. 実際にサンプルコードを動かしてみる
>2\. 自分のプロジェクトにLuciaを導入してみる  
>3\. OAuth2認証（Google、GitHub）の追加を検討する
>4\. セッション管理の詳細をカスタマイズする

Luciaは学習コストが低く、TypeScriptとの相性も抜群なので、認証機能が必要なプロジェクトではぜひ検討してみてくださいね。シンプルながら実務で使える機能が揃っているので、開発効率が大幅に向上するはずです！