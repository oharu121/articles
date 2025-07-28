# はじめに

本記事では、**TypeScript + Prisma** を使って、Node.jsアプリケーションのデータベース管理を型安全かつ効率的に行う方法を紹介します。

TypeScriptの静的型チェックと、Prismaの直感的なデータモデリング・クエリ機能を組み合わせることで、**開発速度の向上**と**実行時エラーの削減**が期待できます。

**🛠️ 必要な環境**

* Node.js（v16以上推奨）
* TypeScriptの基本的な知識
* PostgreSQLやMySQLなど、任意のRDBMSへのアクセス権

# 1. プロジェクトのセットアップ

- プロジェクト作成と初期化

```bash
mkdir my-prisma-ts-app
cd my-prisma-ts-app
npm init -y
```

- TypeScriptのセットアップ

TypeScript環境が未構築の場合は、以下の記事を参考にしてください：

👉 [TypeScript 開発環境のセットアップ手順（Qiita）](https://qiita.com/oharu121/items/3aadafdd5daa8dc53c64)

- Prisma の導入

```bash
npm install prisma --save-dev
npm install @prisma/client
npx prisma init
```

以下のような構成が生成されます：

```
my-prisma-ts-app/
├── prisma/
│   └── schema.prisma ← スキーマ定義ファイル
└── .env              ← DB接続文字列など
```

# 2. データベーススキーマの定義

例として、`User` モデルを定義：

```prisma
model User {
  id    Int    @id @default(autoincrement())
  name  String
  email String @unique
}
```

`.env` ファイルに接続情報を記述（例: PostgreSQL）：

```env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
```

# 3. Prisma Migrateでマイグレーション

```bash
npx prisma migrate dev --name init
```

これによりマイグレーション履歴が作成され、データベースが `User` テーブルで初期化されます。

# 4. PrismaClientのSingleton化

PrismaClientはインスタンスごとにDB接続を生成するため、**毎回 new PrismaClient() すると接続が無駄に増えます**。特に開発中やサーバレス環境では深刻な問題になります。

そこで、**Singletonパターン**で1インスタンスだけを使い回す方法が推奨されます。

- 📁 `lib/prisma.ts` を作成

```ts:lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: ['query'], // 必要に応じて省略
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

✅ これでHot Reload時の再接続を防げます。

# 5. Prismaを使ったデータ操作例

```ts:index.t
import { prisma } from './lib/prisma';

async function main() {
  const newUser = await prisma.user.create({
    data: {
      name: 'Alice',
      email: 'alice@example.com',
    },
  });

  console.log('Created user:', newUser);
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

# まとめ
TypeScriptとPrismaの組み合わせにより、Node.jsアプリケーションのデータベース管理がより安全で効率的になります。このセットアップは、開発プロセスを加速させ、エラーのリスクを最小化するため、特に大規模なアプリケーション開発において非常に有効です
