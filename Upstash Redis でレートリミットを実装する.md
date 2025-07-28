# Upstash Redis でレートリミットを実装する

# はじめに

Web APIなどを公開していると、「短時間に同じユーザーから大量リクエストが来るのを防ぎたい」と思うことはありませんか？

そんな時に便利なのが「**レートリミット**」です。本記事では、**Upstash Redis** を使って、簡単かつスケーラブルにレートリミットを実装する方法を紹介します。

**🧩 なぜレートリミットが必要？**

Gemini API は無料枠があるとはいえ、過剰な呼び出しは課金や制限の原因になります。また、不特定多数のユーザーに公開するアプリでは **悪意ある連打** にも備える必要があります。

**🎯 実現したいこと**

* 同じユーザー（識別子）からのアクセス回数を制限したい
* 制限回数を超えたら処理をブロックしたい
* 毎日リセットされるようにしたい

そこで、今回は `@upstash/redis` を使って簡単に「1ユーザーあたり1日5回まで」という制限を追加してみます。

#  Redisとは？

**Redis（リディス）** は、超高速で動作する **インメモリ型のデータベース** です。

* データをメモリ（RAM）上に保存するため、**非常に高速**に読み書きできる
* 一時的なカウントやキャッシュ、キュー、セッション管理などに広く使われている
* キーと値の組み合わせでデータを保持する「NoSQL」型データベース

**レートリミットのように「回数をカウントして一時的に保持」する用途にぴったり**です。

# Upstashとは？

**Upstash（アップスタッシュ）** は、Redisをクラウド上で提供してくれるマネージドサービスです。

特徴として：

* **VercelやCloudflare Workersなどのサーバレス環境**と相性が良い
* **REST APIでRedisにアクセス**できる（Redisのクライアント接続が不要）
* **無料プランあり** → 個人開発にも嬉しい
* グローバルなエッジネットワークで**低レイテンシ**

簡単に言えば、

> Redisを自分で立ち上げたり管理せずに、Upstashに任せてすぐ使えるようにしたもの

という感じです。

**🔗 RedisとUpstashの関係**

| サービス    | 役割                       |
| ------- | ------------------------ |
| Redis   | 高速なインメモリデータベース（本体）       |
| Upstash | Redisをクラウドで簡単に使えるようにしたもの |

**💡 なぜUpstash Redis？**

* **REST API経由で使える** → VercelやCloudflareなどサーバレス環境でも安心
* **無料枠あり** → 個人開発にも最適
* **自動スケーリング** → 急なアクセス増にも耐えられる

https://upstash.com/

# Upstash Redisのセットアップ

まず Upstash Redis の REST URL と Token を発行し、`.env.local` に保存します。

```env
UPSTASH_REDIS_REST_URL=your_redis_url
UPSTASH_REDIS_REST_TOKEN=your_redis_token
```

インストール：

```bash
npm install @upstash/redis
```

# レートリミット関数の実装

```ts:lib/rateLimit.ts
import { Redis } from '@upstash/redis'

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL || '',
  token: process.env.UPSTASH_REDIS_REST_TOKEN || ''
})

/**
 * レートリミットをチェックする関数
 * @param identifier IPやユーザーIDなど、識別子
 * @returns 制限以内なら true、超えていれば false
 */
export async function checkRateLimit(identifier: string): Promise<boolean> {
  const key = `rate_limit:${identifier}`;
  const limit = 5; // 1日あたりの最大回数

  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.expire(key, 60 * 60 * 24); // 24時間後に期限切れ
  }

  return current <= limit;
}
```

**🔍 解説**
* `identifier` は `IP` や `userId`、`email` などに置き換えて使えます。

* `redis.incr(key)`
  Redisで指定キーの値をインクリメントします。キーがなければ `1` に初期化されます。

* `redis.expire(key, seconds)`
  有効期限を秒単位で指定。ここでは24時間（`24 * 60 * 60`）でキーが自動削除されます。

* `current <= limit`
  現在のアクセス数が制限以内かをチェックします。

**📌 補足：`identifier`（識別子）をどう設計するか？**

| 識別子     | メリット            | デメリット          |
| ------- | --------------- | -------------- |
| IPアドレス  | 実装が簡単、匿名ユーザーでも可 | 同一ネットワークで共有される |
| ユーザーID  | ログイン済みなら正確      | 匿名ユーザーには使えない   |
| メールアドレス | 複数端末をまたいでも制御可能  | 利用には認証が必要      |

# Gemini 関数との統合例

```ts:pages/api/generate.ts
import type { NextApiRequest, NextApiResponse } from 'next'
import { generateWithGemini } from '@/lib/gemini'
import { checkRateLimit } from '@/lib/rateLimit'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const prompt = req.body.prompt
  const ip = req.headers['x-forwarded-for'] || req.socket.remoteAddress || 'unknown'

  const allowed = await checkRateLimit(String(ip))

  if (!allowed) {
    return res.status(429).json({ error: 'Rate limit exceeded. Try again tomorrow.' })
  }

  try {
    const result = await generateWithGemini({ prompt })
    res.status(200).json({ result })
  } catch (error) {
    res.status(500).json({ error: 'Failed to generate content' })
  }
}
```

# おわりに

本記事では、**Node.js + Upstash Redis** を使ったシンプルなレートリミットの実装方法を紹介しました。

APIの保護やスパム防止に非常に有効なので、ぜひ導入してみてください！
