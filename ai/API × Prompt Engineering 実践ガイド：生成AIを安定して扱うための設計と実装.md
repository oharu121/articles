# はじめに

生成AIをAPI経由で使う際、最も重要なのが **プロンプトの設計** です。
どれだけ高性能なモデルを使っていても、プロンプトが曖昧であれば、出力も不安定になります。

本記事では、**Web APIで生成AIを扱う開発者向けに**、プロンプト設計の基本から具体例、注意点までを整理して紹介します。

以下のGithubレポジトリーをもとに、この記事を書きました。

https://github.com/yuting0624/newYear2025

**🧩 なぜプロンプト設計が重要なのか？**

生成AIは **「入力された文章」=プロンプト** をもとに応答を生成します。
とくにAPIでは、以下のような特性が要求されます：

| 要求される特性 | 内容                     |
| ------- | ---------------------- |
| 一貫性     | 毎回似た出力を得たい（テンプレート的運用）  |
| 安定性     | フォーマットがブレるとフロント側で扱いづらい |
| 保守性     | テキスト修正で挙動が変わるので見通しが必要  |

**🎯 この記事でわかること**

* Web APIで生成AIを扱うための**プロンプト設計のベストプラクティス**
* 実運用で役立つ**レート制限（Rate Limit）実装**
* 生成結果の**永続化（Redis保存）**
* Gemini APIとの連携方法（`generateWithGemini`）

**🧱 使用技術スタック**

* フレームワーク：Next.js（App Router / Edge Runtime）
* AIモデル：Google Gemini API（`@google/generative-ai`）
* ストレージ・制限：Upstash Redis（REST API）

# プロンプト設計の基本

**実用例①：FAQ自動応答API**

> **目標**：ユーザーからの自然文の質問に対し、企業のFAQ情報に基づいて自動回答を返す。

```ts
const systemPrompt = `あなたはFAQに基づいて自動応答を返すAIアシスタントです。

以下のFAQを参照し、ユーザーの質問に対して正確かつ簡潔に回答してください。
答えがない場合は「その件については担当者におつなぎします」と回答してください。

出力は以下のJSON形式で返してください：

{
  "answer": "[回答文]",
  "confidence": "[高/中/低]"
}

以下がFAQです：
Q: 商品はいつ届きますか？
A: 通常、2〜3営業日以内に発送いたします。

Q: 返品は可能ですか？
A: 商品到着後7日以内なら可能です。

【ユーザーからの質問】
${userQuestion}
`;
```

**実用例②：メール返信文生成API**

> **目標**：受信メールの内容を元に返信文の下書きをAIに作らせたい。敬語やトーンも揃えたい。

```ts
const systemPrompt = `あなたは日本語ビジネスメールのプロフェッショナルです。
以下の受信メールに対して、適切な敬語・トーンを使って自然な返信文を生成してください。
文末には「何卒よろしくお願いいたします。」を入れてください。

【受信メール】
${incomingMail}
`;
```

**実用例③：おみくじ体験API**

> **目標**：ユーザーがボタンを押すと、AIが日本の神社でおみくじを引いたような体験を返す。

```ts
const systemPrompt = `あなたは日本の神社でおみくじを引く体験を提供するAIです
以下のルールを守り、ユーザーの名前と目標に合わせた内容を親しみやすく出力してください。

出力ルール：
- 出力は簡潔かつ明るく、絵文字を活用すること
- 以下のフォーマットに厳密に従うこと

 出力フォーマット：
🧧 運勢：［大吉／吉／中吉／小吉／末吉／凶］

🙏 願い事：［ユーザーの目標に関連した具体的な内容］
💼 仕事：［ユーザーの目標に関連した具体的なアドバイス］
📚 学問：［ユーザーの目標に関連した具体的なアドバイス］
💘 恋愛：［ユーザーの目標に関連した具体的なアドバイス］
💪 健康：［ユーザーの目標に関連した具体的なアドバイス］

🌈 総合アドバイス：［全体的な助言と励まし］

出力の最後には、ユーザーを前向きに励ます短いメッセージを添えてください。
`;
```

**⚠️ APIでのプロンプト設計の注意点**

| 項目         | 内容                                 |
| ---------- | ---------------------------------- |
| ✅ 役割の明示   | 「あなたは〇〇なAIです」などで出力の一貫性を確保          |
| ✅ 出力形式の固定 | JSON / Markdown / 箇条書き などフォーマットを明示 |
| ✅ 例外処理の定義 | 「該当しない場合は 'わかりません' と返す」などを指定       |
| ❌ 曖昧な表現を避ける       | 「できるだけ簡潔に」だけではAIが曖昧な挙動をすることも    |
| ❌ 長すぎるプロンプト       | 入力上限やコストに注意（特にRAG併用時）           |

# APIの実装

### 1. `generateWithGemini` のラッパー関数

```ts
import { GoogleGenerativeAI } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || '');

export async function generateWithGemini(prompt: string): Promise<string> {
  try {
    const model = genAI.getGenerativeModel({ model: 'gemini-pro' });
    const result = await model.generateContent(prompt);
    const response = result.response;
    return response.text();
  } catch (err) {
    console.error('Gemini Error:', err);
    throw new Error('生成に失敗しました');
  }
}
```

**✅ 解説**

* モデルは `gemini-pro`
* 失敗時は例外を投げて呼び出し元で `500` 応答を返す
* promptは**完全な文字列**として渡す（テンプレ化推奨）

詳細な紹介はこちら：

https://zenn.dev/oharu121/articles/b9ed1217653778

### 2. Upstash Redisによるレート制限

```ts
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export async function checkRateLimit(identifier: string): Promise<boolean> {
  const key = `rate_limit:${identifier}`;
  const limit = 5;

  const current = await redis.incr(key);
  if (current === 1) {
    await redis.expire(key, 24 * 60 * 60); // 24時間
  }

  return current <= limit;
}
```

**✅ 解説**

* 1ユーザーあたり**日5回制限**
* `identifier` はユーザーのIPやUUIDなど
* `incr + expire` でトークンバケット風に実現

詳細な紹介はこちら：

https://zenn.dev/oharu121/articles/0c15661bc418ff

### 3. 生成結果の保存（Redisへの永続化）

```ts
interface LogData {
  prompt: string;
  result: string;
  timestamp: string;
}

await redis.lpush('generation_logs', JSON.stringify({
  prompt,
  result,
  timestamp: new Date().toISOString()
}));
```

**✅ 解説**

* `lpush` によってRedisに**最新順ログ**を保存
* 今後**分析・UI表示**にも使える
* Redisのキーは用途別に切る（例：`generation_logs`）

**API全体の構成（POSTハンドラ）**

```ts
export async function POST(req: Request) {
  try {
    const { prompt, identifier } = await req.json();

    const isAllowed = await checkRateLimit(identifier);
    
    if (!isAllowed) {
      return NextResponse.json({ error: '回数制限に達しました' }, { status: 429 });
    }

    const systemPrompt = '...'; // 上で紹介したテンプレート
    const fullPrompt = `${systemPrompt}\n\n${prompt}`;
    const result = await generateWithGemini(fullPrompt);

    await redis.lpush('generation_logs', JSON.stringify({
      prompt,
      result,
      timestamp: new Date().toISOString()
    }));

    return NextResponse.json({ result });
  } catch (err) {
    return NextResponse.json({ error: '生成エラー' }, { status: 500 });
  }
}
```

**📚 まとめ**

| パート         | ポイント                       |
| ----------- | -------------------------- |
| 🧠 プロンプト設計  | 役割・出力形式・例外対応を明記せよ          |
| 🤖 Gemini連携 | generateWithGemini で安定的に使う |
| 🔒 レート制限    | checkRateLimitで abuse防止    |
| 💾 ログ保存     | lpushで将来のUX改善に備える          |

# おわりに

プロンプトは「指示書」であり、「UI」であり、「設計図」です。
APIを通してAIを使うなら、**プロンプトを設計する力こそがプロダクトの完成度を決めます。**

**✅ 明日から使えるプロンプト設計3ステップ**

1. **役割とトーンを定義**
2. **出力形式をテンプレート化**
3. **想定外の入力への対応も指示**
