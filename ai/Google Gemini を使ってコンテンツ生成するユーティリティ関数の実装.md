# Google Gemini を使ってコンテンツ生成するユーティリティ関数の実装

# はじめに

最近話題の Google の生成AI「Gemini」。公式の npm パッケージ [`@google/generative-ai`](https://www.npmjs.com/package/@google/generative-ai) を使えば、Node.js や Next.js プロジェクトでも簡単に扱えます。

本記事では、Gemini API を使ってプロンプトからテキストを生成する関数 `generateWithGemini` を実装してみます。

# セットアップ

```bash
npm install @google/generative-ai
```

`.env` に API キーを設定：

```env
GEMINI_API_KEY=your_google_api_key
```

# 実装コード

```ts
// lib/gemini.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || '');

export async function generateWithGemini(prompt: string) {
  try {
    const model = genAI.getGenerativeModel({ model: "gemini-2.0-flash-exp" });

    const generationConfig = {
      temperature: 0.7,
      topK: 40,
      topP: 0.9,
      maxOutputTokens: 1024,
    };

    const result = await model.generateContent({
      contents: [{ role: "user", parts: [{ text: prompt }] }],
      generationConfig,
    });

    const response = await result.response;
    return response.text();
  } catch (error) {
    console.error('Error in generateWithGemini:', error);
    throw error;
  }
}
```

**🧭 補足解説**

| パラメータ             | 説明                                                                           |
| ----------------- | ---------------------------------------------------------------------------- |
| `temperature`     | ランダム性。0.0 に近いとより決定的、1.0 に近いとより多様性あり。                                         |
| `topK` / `topP`   | 単語選択のサンプリング手法。生成の多様性に影響します。                                                  |
| `maxOutputTokens` | 出力される最大トークン数。                                                                |
| `model`           | 使用する Gemini モデル。例: `gemini-pro`, `gemini-2.0-pro`, `gemini-2.0-flash-exp` など |

**✅ 使用例**

```ts
const text = await generateWithGemini("ReactでTodoアプリを作る手順を教えて");
console.log(text);
```

# おわりに

今回は Google Gemini を Node.js プロジェクトで利用するための基本的なユーティリティ関数を紹介しました。OpenAI API に慣れている人にとっても扱いやすく、生成系のタスクに役立ちます。今後は画像生成やマルチモーダル対応にもチャレンジしていきたいですね！
