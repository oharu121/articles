# Drizzle ORMで始める型安全なデータベース操作：軽量・高速なPrisma代替の実装ガイド

## はじめに

**Drizzle ORM**は、TypeScriptファーストな軽量ORMとして注目を集めています。Prismaと比較して**バンドルサイズが小さく**、**実行時のオーバーヘッドが少ない**という特徴があり、特にエッジ環境やサーバーレス関数での使用に適しています。

本記事では、Drizzle ORMを使って型安全なデータベース操作を実装する方法を、**実際のコード例を交えて**詳しく解説します。

🎯 **この記事で学べること**

>* Drizzle ORMの基本的なセットアップ方法
>* スキーマ定義とマイグレーションの実践
>* 型安全なCRUD操作の実装
>* リレーション設計とクエリの最適化
>* 実際のアプリケーションでの活用例

💡 **想定読者**

>* TypeScriptでのバックエンド開発経験がある方
>* ORMを使ったデータベース操作に興味がある方  
>* Prisma以外の選択肢を検討している方

## Drizzle ORMの特徴とメリット

**🛒 Prismaとの比較**

| 項目 | Drizzle ORM | Prisma |
|------|-------------|--------|
| **バンドルサイズ** | 軽量（~50KB） | 重い（~2MB） |
| **実行時性能** | 高速（直接SQL生成） | やや重い（Query Engine経由） |
| **型安全性** | 完全な型推論 | 完全な型推論 |
| **学習コスト** | 低い（SQLライク） | 中程度 |
| **エコシステム** | 新しいが活発 | 成熟している |

**✅ Drizzleが適している場面**

>* **エッジ環境**（Cloudflare Workers、Deno Deploy）  
>* **サーバーレス関数**（Vercel Functions、AWS Lambda）  
>* **軽量なアプリケーション**（シンプルなAPI、マイクロサービス）  
>* **高いパフォーマンス**が求められるアプリケーション

## プロジェクトのセットアップ

### 1\. 基本的な環境構築

```bash
# プロジェクト作成
mkdir drizzle-example
cd drizzle-example
npm init -y

# 必要なパッケージのインストール
npm install drizzle-orm
npm install -D drizzle-kit
npm install better-sqlite3
npm install -D @types/better-sqlite3 tsx typescript
```

### 2\. プロジェクト構成

```
drizzle-example/
├── src/
│   ├── db/
│   │   ├── schema.ts    # スキーマ定義
│   │   ├── index.ts     # DB接続とクライアント
│   │   └── migrate.ts   # マイグレーション実行
│   └── index.ts         # メインロジック
├── drizzle/             # マイグレーションファイル
├── drizzle.config.ts    # Drizzle設定
└── package.json
```

### 3\. TypeScript設定

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## スキーマ定義とマイグレーション

### 1\. データベーススキーマの作成

```typescript:src/db/schema.ts
import { sqliteTable, text, integer, real } from 'drizzle-orm/sqlite-core';
import { relations } from 'drizzle-orm';

// ユーザーテーブル
export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  avatar: text('avatar'),
  createdAt: integer('created_at', { mode: 'timestamp' })
    .notNull()
    .$defaultFn(() => new Date()),
});

// 投稿テーブル
export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  content: text('content').notNull(),
  authorId: integer('author_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  published: integer('published', { mode: 'boolean' }).default(false),
  createdAt: integer('created_at', { mode: 'timestamp' })
    .notNull()
    .$defaultFn(() => new Date()),
});

// コメントテーブル
export const comments = sqliteTable('comments', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  content: text('content').notNull(),
  postId: integer('post_id')
    .notNull()
    .references(() => posts.id, { onDelete: 'cascade' }),
  authorId: integer('author_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  createdAt: integer('created_at', { mode: 'timestamp' })
    .notNull()
    .$defaultFn(() => new Date()),
});

// リレーション定義
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
  comments: many(comments),
}));

export const postsRelations = relations(posts, ({ one, many }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
  comments: many(comments),
}));

export const commentsRelations = relations(comments, ({ one }) => ({
  post: one(posts, {
    fields: [comments.postId],
    references: [posts.id],
  }),
  author: one(users, {
    fields: [comments.authorId],
    references: [users.id],
  }),
}));

// 型定義をエクスポート
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
export type Post = typeof posts.$inferSelect;
export type NewPost = typeof posts.$inferInsert;
export type Comment = typeof comments.$inferSelect;
export type NewComment = typeof comments.$inferInsert;
```

### 2\. データベース接続の設定

```typescript:src/db/index.ts
import Database from 'better-sqlite3';
import { drizzle } from 'drizzle-orm/better-sqlite3';
import * as schema from './schema';

// SQLiteデータベースの初期化
const sqlite = new Database('app.db');

// Drizzleインスタンスの作成
export const db = drizzle(sqlite, { schema });

// 型安全なクエリビルダーをエクスポート
export { schema };
```

### 3\. Drizzle設定ファイル

```typescript:drizzle.config.ts
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './drizzle',
  driver: 'better-sqlite',
  dbCredentials: {
    url: './app.db'
  }
} satisfies Config;
```

### 4\. マイグレーションの実行

```bash
# マイグレーションファイル生成
npx drizzle-kit generate:sqlite

# マイグレーション実行
npx drizzle-kit push:sqlite
```

## 型安全なCRUD操作の実装

### 1\. 基本的なデータ操作クラス

```typescript:src/services/userService.ts
import { eq, like, desc } from 'drizzle-orm';
import { db, schema } from '../db';
import type { NewUser, User } from '../db/schema';

export class UserService {
  // ユーザー作成
  async createUser(userData: NewUser): Promise<User> {
    try {
      const [user] = await db
        .insert(schema.users)
        .values(userData)
        .returning();
      
      return user;
    } catch (error) {
      if (error.code === 'SQLITE_CONSTRAINT_UNIQUE') {
        throw new Error('このメールアドレスは既に使用されています');
      }
      throw error;
    }
  }

  // ユーザー一覧取得（ページネーション付き）
  async getUsers(page = 1, limit = 10): Promise<User[]> {
    const offset = (page - 1) * limit;
    
    return await db
      .select()
      .from(schema.users)
      .orderBy(desc(schema.users.createdAt))
      .limit(limit)
      .offset(offset);
  }

  // ID別ユーザー取得（投稿とコメント含む）
  async getUserWithPosts(userId: number) {
    return await db.query.users.findFirst({
      where: eq(schema.users.id, userId),
      with: {
        posts: {
          orderBy: desc(schema.posts.createdAt),
          with: {
            comments: {
              orderBy: desc(schema.comments.createdAt),
              with: {
                author: true
              }
            }
          }
        }
      }
    });
  }

  // ユーザー検索
  async searchUsers(query: string): Promise<User[]> {
    return await db
      .select()
      .from(schema.users)
      .where(like(schema.users.name, `%${query}%`))
      .orderBy(desc(schema.users.createdAt));
  }

  // ユーザー更新
  async updateUser(id: number, updates: Partial<NewUser>): Promise<User | null> {
    const [updatedUser] = await db
      .update(schema.users)
      .set({
        ...updates,
        // 更新日時を自動設定したい場合
        // updatedAt: new Date()
      })
      .where(eq(schema.users.id, id))
      .returning();
    
    return updatedUser || null;
  }

  // ユーザー削除
  async deleteUser(id: number): Promise<boolean> {
    const result = await db
      .delete(schema.users)
      .where(eq(schema.users.id, id));
    
    return result.changes > 0;
  }
}
```

### 2\. 投稿管理サービス

```typescript:src/services/postService.ts
import { eq, desc, and } from 'drizzle-orm';
import { db, schema } from '../db';
import type { NewPost, Post } from '../db/schema';

export class PostService {
  // 投稿作成
  async createPost(postData: NewPost): Promise<Post> {
    const [post] = await db
      .insert(schema.posts)
      .values(postData)
      .returning();
    
    return post;
  }

  // 公開済み投稿一覧取得
  async getPublishedPosts(limit = 10): Promise<any[]> {
    return await db.query.posts.findMany({
      where: eq(schema.posts.published, true),
      orderBy: desc(schema.posts.createdAt),
      limit,
      with: {
        author: {
          columns: {
            id: true,
            name: true,
            avatar: true
          }
        },
        comments: {
          orderBy: desc(schema.comments.createdAt),
          limit: 5,
          with: {
            author: {
              columns: {
                id: true,
                name: true,
                avatar: true
              }
            }
          }
        }
      }
    });
  }

  // 投稿詳細取得
  async getPostById(id: number) {
    return await db.query.posts.findFirst({
      where: eq(schema.posts.id, id),
      with: {
        author: true,
        comments: {
          orderBy: desc(schema.comments.createdAt),
          with: {
            author: true
          }
        }
      }
    });
  }

  // ユーザーの投稿一覧
  async getUserPosts(authorId: number): Promise<Post[]> {
    return await db
      .select()
      .from(schema.posts)
      .where(eq(schema.posts.authorId, authorId))
      .orderBy(desc(schema.posts.createdAt));
  }

  // 投稿の公開状態変更
  async togglePublishStatus(id: number, published: boolean): Promise<Post | null> {
    const [post] = await db
      .update(schema.posts)
      .set({ published })
      .where(eq(schema.posts.id, id))
      .returning();
    
    return post || null;
  }
}
```

## 実際の使用例とベストプラクティス

### 1\. メインアプリケーションの実装

```typescript:src/index.ts
import { UserService } from './services/userService';
import { PostService } from './services/postService';

async function main() {
  const userService = new UserService();
  const postService = new PostService();

  try {
    // 1. ユーザー作成
    console.log('🔄 ユーザーを作成中...');
    const newUser = await userService.createUser({
      name: '山田太郎',
      email: 'yamada@example.com',
      avatar: 'https://example.com/avatar.jpg'
    });
    console.log('✅ ユーザー作成完了:', newUser);

    // 2. 投稿作成
    console.log('🔄 投稿を作成中...');
    const newPost = await postService.createPost({
      title: 'Drizzle ORMを使ってみた感想',
      content: '軽量で高速なORMとして、Drizzleは素晴らしい選択肢だと思います。',
      authorId: newUser.id,
      published: true
    });
    console.log('✅ 投稿作成完了:', newPost);

    // 3. ユーザー詳細取得（投稿含む）
    console.log('🔄 ユーザー詳細を取得中...');
    const userWithPosts = await userService.getUserWithPosts(newUser.id);
    console.log('✅ ユーザー詳細:', JSON.stringify(userWithPosts, null, 2));

    // 4. 公開済み投稿一覧取得
    console.log('🔄 公開投稿一覧を取得中...');
    const publishedPosts = await postService.getPublishedPosts(5);
    console.log('✅ 公開投稿一覧:', publishedPosts.length, '件');

  } catch (error) {
    console.error('❌ エラーが発生しました:', error);
  }
}

main();
```

### 2\. エラーハンドリングと型安全性

```typescript:src/utils/errorHandler.ts
import { DrizzleError } from 'drizzle-orm';

export class DatabaseError extends Error {
  constructor(
    message: string,
    public code?: string,
    public cause?: unknown
  ) {
    super(message);
    this.name = 'DatabaseError';
  }
}

export function handleDrizzleError(error: unknown): never {
  if (error instanceof DrizzleError) {
    throw new DatabaseError(
      'データベース操作中にエラーが発生しました',
      error.message,
      error
    );
  }
  
  if (error && typeof error === 'object' && 'code' in error) {
    const sqliteError = error as { code: string; message: string };
    
    switch (sqliteError.code) {
      case 'SQLITE_CONSTRAINT_UNIQUE':
        throw new DatabaseError('重複したデータが存在します');
      case 'SQLITE_CONSTRAINT_FOREIGNKEY':
        throw new DatabaseError('関連するデータが存在しません');
      default:
        throw new DatabaseError(
          `データベースエラー: ${sqliteError.message}`
        );
    }
  }
  
  throw error;
}
```

## パフォーマンス最適化のポイント

### 1\. インデックスの設定

```typescript:src/db/schema.ts
// インデックス付きのスキーマ例
export const posts = sqliteTable('posts', {
  // ... 既存のカラム定義
}, (table) => ({
  // 複合インデックス
  authorPublishedIdx: index('author_published_idx')
    .on(table.authorId, table.published),
  // 作成日時インデックス
  createdAtIdx: index('created_at_idx').on(table.createdAt),
}));
```

### 2\. 準備済みステートメントの活用

```typescript:src/services/optimizedQueries.ts
import { eq } from 'drizzle-orm';
import { db, schema } from '../db';

// 準備済みステートメントでパフォーマンス向上
export class OptimizedQueries {
  // よく使用されるクエリは準備済みステートメントに
  private static getUserByIdStatement = db
    .select()
    .from(schema.users)
    .where(eq(schema.users.id, $userId))
    .prepare();

  static async getUserById(id: number) {
    return await this.getUserByIdStatement.execute({ userId: id });
  }
}
```

## まとめ

Drizzle ORMは、**軽量性**と**型安全性**を両立したモダンなORMです。特にエッジ環境やパフォーマンスが重要なアプリケーションでは、Prismaの優れた代替選択肢となります。

🎯 **Drizzle ORMの主な利点**

>* **軽量**：小さなバンドルサイズでアプリケーションを高速に
>* **型安全**：完全なTypeScript統合で開発時エラーを防止
>* **直感的**：SQLライクな書き方で学習コストを最小化
>* **高性能**：直接SQL生成による高速なクエリ実行

💡 **次のステップ**

>* 実際のプロジェクトでDrizzle ORMを導入してみる
>* 他のデータベース（PostgreSQL、MySQL）との組み合わせを試す
>* 複雑なリレーションやクエリの最適化手法を学ぶ
>* マイグレーション戦略とCI/CD統合を検討する

Drizzle ORMで、より効率的で安全なデータベース操作を実現してください！