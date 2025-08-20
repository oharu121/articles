## MastraでTypeScriptプロジェクトにAIエージェントを組み込む実装ガイド

## はじめに

「AIエージェントを自分のアプリケーションに組み込んでみたい」と思ったことはありませんか？

> 「どのフレームワークを選べばいいの？」
> 「TypeScriptで実用的なAIエージェントって作れるの？」
> 「本番環境で安定して動作させるには？」

こんな悩みを抱えている開発者におすすめしたいのが **Mastra** です。MastraはGatsbyチームが開発したTypeScriptネイティブのAIエージェントフレームワークで、プロダクション対応の強力な機能を提供します。

本記事では、**Mastraを使って実際にTypeScriptプロジェクトにAIエージェントを組み込む方法**を、コード例とともに詳しく解説します。

## Mastraとは？

Mastraは、**TypeScriptでAIエージェントとワークフローを構築するためのフレームワーク**です。Gatsbyの開発チームが手がけており、エンタープライズグレードの機能を重視して設計されています。

| 機能 | 説明 | メリット |
|------|------|--------|
| **Agents** | AIモデルとツールを組み合わせた自律システム | 複雑なタスクの自動実行 |
| **Workflows** | 状態を持つマルチステップ処理 | 複雑な業務フローの自動化 |
| **RAG** | ベクトルストアとの統合 | 専門知識ベースの活用 |
| **Observability** | OpenTelemetryトレーシング | 本番環境での監視・デバッグ |

## プロジェクトのセットアップ

### 1. インストールと初期設定

まず、Mastraプロジェクトを作成しましょう。

```bash
# 新しいMastraプロジェクトを作成
npm create mastra@latest my-ai-project
cd my-ai-project

# 必要な依存関係をインストール
npm install
```

### 2. 環境変数の設定

`.env`ファイルでAPI キーを設定します。

```env:.env
OPENAI_API_KEY=your_openai_api_key
ANTHROPIC_API_KEY=your_anthropic_api_key
```

### 3. 基本的なプロジェクト構造

```
my-ai-project/
├── src/
│   ├── agents/          # エージェント定義
│   ├── workflows/       # ワークフロー
│   ├── tools/          # カスタムツール
│   └── index.ts        # エントリーポイント
├── mastra.config.js    # Mastra設定
└── package.json
```

## 基本的なエージェントの実装

### 1. シンプルなエージェントの作成

最初に、基本的なエージェントを作成してみましょう。

```typescript:src/agents/article-agent.ts
import { Agent } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

// エージェントの設定
const articleAgent = new Agent({
  name: 'ArticleAgent',
  instructions: `
    あなたは技術記事を作成する専門家です。
    与えられたトピックについて、初心者にもわかりやすく
    実践的な記事を書いてください。
  `,
  model: openai('gpt-4o-mini'),
});

// エージェントを使った記事生成
export async function generateArticle(topic: string) {
  try {
    const result = await articleAgent.text(`
      「${topic}」について、以下の構成で技術記事を作成してください：
      
      1. はじめに（問題の提起）
      2. 基本概念の説明
      3. 実装例（コード付き）
      4. よくある課題と解決方法
      5. まとめ
      
      読者は中級レベルのTypeScript開発者を想定してください。
    `);
    
    return result.text;
  } catch (error) {
    console.error('記事生成エラー:', error);
    throw error;
  }
}
```

### 2. メモリ機能付きエージェント

対話の履歴を記憶するエージェントを作成します。

```typescript:src/agents/conversational-agent.ts
import { Agent, Memory } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

// メモリ設定
const memory = new Memory({
  name: 'chat-memory',
  store: 'local' // または 'redis', 'file'
});

const chatAgent = new Agent({
  name: 'ChatAssistant',
  instructions: `
    あなたは親切なプログラミングアシスタントです。
    ユーザーとの会話履歴を覚えていて、
    前の文脈を考慮して回答してください。
  `,
  model: openai('gpt-4o-mini'),
  memory,
});

export async function chatWithAgent(message: string, sessionId: string) {
  const response = await chatAgent.text(message, {
    sessionId, // セッションIDでメモリを管理
  });
  
  return response.text;
}

// 使用例
export async function runConversation() {
  const sessionId = 'user-123';
  
  // 最初の質問
  const response1 = await chatWithAgent(
    'TypeScriptでAPIクライアントを作る方法を教えて',
    sessionId
  );
  console.log('回答1:', response1);
  
  // フォローアップ質問（前の文脈を覚えている）
  const response2 = await chatWithAgent(
    'さっきの方法でエラーハンドリングはどうすればいい？',
    sessionId
  );
  console.log('回答2:', response2);
}
```

## カスタムツールの実装

### 1. ファイル操作ツール

エージェントがファイルシステムを操作できるツールを作成します。

```typescript:src/tools/file-tool.ts
import { Tool } from '@mastra/core';
import { promises as fs } from 'fs';
import path from 'path';

export const fileOperationTool: Tool = {
  id: 'file-operations',
  name: 'File Operations',
  description: 'ファイルの作成、読み取り、削除を行う',
  schema: {
    type: 'object',
    properties: {
      operation: {
        type: 'string',
        enum: ['create', 'read', 'delete', 'list'],
        description: '実行する操作'
      },
      filePath: {
        type: 'string',
        description: 'ファイルパス'
      },
      content: {
        type: 'string',
        description: 'ファイルに書き込む内容（create時）'
      }
    },
    required: ['operation', 'filePath']
  },
  
  execute: async ({ operation, filePath, content }) => {
    try {
      switch (operation) {
        case 'create':
          await fs.writeFile(filePath, content || '', 'utf-8');
          return { success: true, message: `ファイル ${filePath} を作成しました` };
          
        case 'read':
          const data = await fs.readFile(filePath, 'utf-8');
          return { success: true, content: data };
          
        case 'delete':
          await fs.unlink(filePath);
          return { success: true, message: `ファイル ${filePath} を削除しました` };
          
        case 'list':
          const dir = path.dirname(filePath);
          const files = await fs.readdir(dir);
          return { success: true, files };
          
        default:
          throw new Error(`未対応の操作: ${operation}`);
      }
    } catch (error) {
      return { 
        success: false, 
        error: error instanceof Error ? error.message : '不明なエラー' 
      };
    }
  }
};
```

### 2. Web API呼び出しツール

外部APIと連携するツールを作成します。

```typescript:src/tools/api-tool.ts
import { Tool } from '@mastra/core';
import axios from 'axios';

export const apiRequestTool: Tool = {
  id: 'api-request',
  name: 'API Request',
  description: 'Web APIへのHTTPリクエストを実行する',
  schema: {
    type: 'object',
    properties: {
      method: {
        type: 'string',
        enum: ['GET', 'POST', 'PUT', 'DELETE'],
        description: 'HTTPメソッド'
      },
      url: {
        type: 'string',
        description: 'リクエスト先URL'
      },
      headers: {
        type: 'object',
        description: 'リクエストヘッダー'
      },
      data: {
        type: 'object',
        description: 'リクエストボディ'
      }
    },
    required: ['method', 'url']
  },
  
  execute: async ({ method, url, headers, data }) => {
    try {
      const response = await axios({
        method: method.toLowerCase(),
        url,
        headers,
        data,
        timeout: 10000 // 10秒でタイムアウト
      });
      
      return {
        success: true,
        status: response.status,
        data: response.data,
        headers: response.headers
      };
    } catch (error) {
      if (axios.isAxiosError(error)) {
        return {
          success: false,
          error: error.message,
          status: error.response?.status,
          data: error.response?.data
        };
      }
      
      return {
        success: false,
        error: error instanceof Error ? error.message : '不明なエラー'
      };
    }
  }
};
```

### 3. ツール付きエージェントの使用

作成したツールをエージェントに組み込みます。

```typescript:src/agents/tool-agent.ts
import { Agent } from '@mastra/core';
import { openai } from '@mastra/provider-openai';
import { fileOperationTool } from '../tools/file-tool';
import { apiRequestTool } from '../tools/api-tool';

const toolAgent = new Agent({
  name: 'ToolAgent',
  instructions: `
    あなたはファイル操作とAPI呼び出しができるアシスタントです。
    ユーザーのリクエストに応じて適切なツールを使用してください。
  `,
  model: openai('gpt-4o-mini'),
  tools: [fileOperationTool, apiRequestTool]
});

// ファイル操作の例
export async function manageFiles() {
  const result = await toolAgent.text(`
    以下の作業を実行してください：
    1. 'output' ディレクトリに 'sample.txt' ファイルを作成
    2. ファイルに "Hello, Mastra!" という内容を書き込み
    3. 作成されたファイルの内容を確認
  `);
  
  return result.text;
}

// API呼び出しの例
export async function fetchExternalData() {
  const result = await toolAgent.text(`
    JSONPlaceholderのAPIを使って以下を実行してください：
    1. https://jsonplaceholder.typicode.com/posts/1 から投稿データを取得
    2. 取得したデータの内容を要約
  `);
  
  return result.text;
}
```

## ワークフローの実装

### 1. 基本的なワークフロー

複数のステップを組み合わせたワークフローを作成します。

```typescript:src/workflows/content-workflow.ts
import { Workflow, Step } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

// リサーチステップ
const researchStep: Step = {
  id: 'research',
  action: async ({ topic }) => {
    const agent = new Agent({
      name: 'Researcher',
      instructions: '指定されたトピックについて詳細な調査を行い、重要なポイントをまとめてください。',
      model: openai('gpt-4o-mini'),
    });
    
    const result = await agent.text(`「${topic}」について、以下の観点から調査してください：
    1. 基本概念と定義
    2. 主要な特徴や利点
    3. 実際の使用例
    4. 注意すべき点やベストプラクティス
    `);
    
    return {
      researchData: result.text
    };
  }
};

// 執筆ステップ
const writingStep: Step = {
  id: 'writing',
  action: async ({ researchData, topic }) => {
    const agent = new Agent({
      name: 'Writer',
      instructions: '技術記事を作成する専門家として、読みやすく実践的な記事を書いてください。',
      model: openai('gpt-4o-mini'),
    });
    
    const result = await agent.text(`
    以下の調査データを基に、「${topic}」についての技術記事を作成してください：
    
    調査データ：
    ${researchData}
    
    記事の要件：
    - マークダウン形式
    - コード例を含める
    - 初心者にもわかりやすい説明
    - 実践的な内容
    `);
    
    return {
      article: result.text
    };
  }
};

// レビューステップ
const reviewStep: Step = {
  id: 'review',
  action: async ({ article }) => {
    const agent = new Agent({
      name: 'Reviewer',
      instructions: '記事の品質をチェックし、改善提案を行う専門家です。',
      model: openai('gpt-4o-mini'),
    });
    
    const result = await agent.text(`
    以下の記事をレビューし、改善提案を行ってください：
    
    ${article}
    
    チェック項目：
    - 内容の正確性
    - 構成のわかりやすさ
    - コード例の品質
    - 改善提案
    `);
    
    return {
      reviewResult: result.text,
      finalArticle: article
    };
  }
};

// ワークフローの定義
export const contentCreationWorkflow = new Workflow({
  name: 'Content Creation Workflow',
  steps: [researchStep, writingStep, reviewStep]
});

// ワークフローの実行
export async function createArticle(topic: string) {
  const result = await contentCreationWorkflow.execute({
    topic
  });
  
  return result;
}
```

### 2. 条件分岐付きワークフロー

条件に応じて処理を分岐するワークフローを作成します。

```typescript:src/workflows/conditional-workflow.ts
import { Workflow, Step } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

// コンテンツ分析ステップ
const analysisStep: Step = {
  id: 'analysis',
  action: async ({ content }) => {
    const agent = new Agent({
      name: 'ContentAnalyzer',
      instructions: 'コンテンツの種類と品質を分析してください。',
      model: openai('gpt-4o-mini'),
    });
    
    const result = await agent.text(`
    以下のコンテンツを分析し、JSON形式で回答してください：
    
    ${content}
    
    分析項目：
    {
      "type": "記事のタイプ（tutorial, reference, etc.）",
      "quality": "品質スコア（1-10）",
      "needsImprovement": true/false,
      "topics": ["関連トピック1", "関連トピック2"]
    }
    `);
    
    try {
      const analysis = JSON.parse(result.text);
      return analysis;
    } catch {
      return {
        type: 'unknown',
        quality: 5,
        needsImprovement: true,
        topics: []
      };
    }
  }
};

// 改善ステップ
const improvementStep: Step = {
  id: 'improvement',
  condition: ({ analysis }) => analysis.needsImprovement,
  action: async ({ content, analysis }) => {
    const agent = new Agent({
      name: 'ContentImprover',
      instructions: 'コンテンツを改善し、より良い品質にしてください。',
      model: openai('gpt-4o-mini'),
    });
    
    const result = await agent.text(`
    以下のコンテンツを改善してください：
    
    現在の内容：
    ${content}
    
    分析結果：
    品質スコア: ${analysis.quality}/10
    タイプ: ${analysis.type}
    
    改善してほしい点：
    - より具体的な例の追加
    - コードサンプルの改良
    - 構成の最適化
    `);
    
    return {
      improvedContent: result.text
    };
  }
};

// 最終化ステップ
const finalizationStep: Step = {
  id: 'finalization',
  action: async ({ content, improvedContent }) => {
    const finalContent = improvedContent || content;
    
    return {
      finalContent,
      processedAt: new Date().toISOString()
    };
  }
};

export const contentProcessingWorkflow = new Workflow({
  name: 'Content Processing Workflow',
  steps: [analysisStep, improvementStep, finalizationStep]
});
```

## RAG（検索拡張生成）の実装

### 1. ベクトルストアの設定

```typescript:src/rag/knowledge-base.ts
import { Agent, VectorStore } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

// ベクトルストアの初期化
const vectorStore = new VectorStore({
  name: 'tech-docs',
  provider: 'pinecone', // または 'qdrant', 'chromadb'
  config: {
    apiKey: process.env.PINECONE_API_KEY,
    environment: process.env.PINECONE_ENVIRONMENT,
    indexName: 'tech-knowledge-base'
  }
});

// RAG対応エージェント
const ragAgent = new Agent({
  name: 'KnowledgeAssistant',
  instructions: `
    あなたは技術ドキュメントの専門家です。
    提供されたコンテキストを基に、正確で詳細な回答を提供してください。
  `,
  model: openai('gpt-4o-mini'),
  memory: new Memory({ name: 'rag-memory' }),
});

// ドキュメントの追加
export async function addDocuments(documents: Array<{
  content: string;
  metadata: Record<string, any>;
}>) {
  for (const doc of documents) {
    await vectorStore.add({
      content: doc.content,
      metadata: doc.metadata
    });
  }
  
  console.log(`${documents.length}件のドキュメントを追加しました`);
}

// 知識ベースを使った質問応答
export async function askWithKnowledge(question: string) {
  // 関連ドキュメントの検索
  const relevantDocs = await vectorStore.similaritySearch(question, {
    limit: 5,
    threshold: 0.7
  });
  
  // コンテキストの構築
  const context = relevantDocs
    .map(doc => `[${doc.metadata.title}]\n${doc.content}`)
    .join('\n\n---\n\n');
  
  // RAGを使った回答生成
  const response = await ragAgent.text(`
    以下のコンテキストを参考に質問に答えてください：
    
    コンテキスト：
    ${context}
    
    質問：
    ${question}
    
    回答の要件：
    - コンテキストの情報を活用する
    - 不足している情報があれば言及する
    - 具体的で実践的な内容にする
  `);
  
  return {
    answer: response.text,
    sources: relevantDocs.map(doc => doc.metadata),
    confidence: relevantDocs.length > 0 ? 'high' : 'low'
  };
}

// 使用例：技術ドキュメントの追加
export async function initializeKnowledgeBase() {
  const docs = [
    {
      content: `
      TypeScriptは、Microsoftによって開発されたプログラミング言語で、
      JavaScriptに静的型付けを追加したものです。
      大規模なアプリケーション開発において、コードの品質と保守性を向上させます。
      `,
      metadata: {
        title: 'TypeScript概要',
        category: 'language',
        difficulty: 'beginner'
      }
    },
    {
      content: `
      Reactは、Facebookによって開発されたJavaScriptライブラリで、
      ユーザーインターフェースの構築に特化しています。
      コンポーネントベースのアーキテクチャが特徴です。
      `,
      metadata: {
        title: 'React基礎',
        category: 'framework',
        difficulty: 'beginner'
      }
    }
  ];
  
  await addDocuments(docs);
}
```

## 実用的なアプリケーション例

### 1. 技術ブログ自動生成システム

```typescript:src/applications/blog-generator.ts
import { Agent, Workflow, Step } from '@mastra/core';
import { openai } from '@mastra/provider-openai';
import { fileOperationTool } from '../tools/file-tool';

class BlogGenerator {
  private researchAgent: Agent;
  private writerAgent: Agent;
  private editorAgent: Agent;
  
  constructor() {
    this.researchAgent = new Agent({
      name: 'ResearchAgent',
      instructions: 'トピックについて徹底的な調査を行い、最新の情報をまとめる専門家',
      model: openai('gpt-4o-mini'),
      tools: [apiRequestTool]
    });
    
    this.writerAgent = new Agent({
      name: 'WriterAgent',
      instructions: '技術記事を魅力的で読みやすい形で執筆する専門家',
      model: openai('gpt-4o-mini'),
      tools: [fileOperationTool]
    });
    
    this.editorAgent = new Agent({
      name: 'EditorAgent',
      instructions: '記事の品質をチェックし、SEO最適化も行う編集者',
      model: openai('gpt-4o-mini'),
    });
  }
  
  async generateBlogPost(topic: string, targetAudience: string = '中級開発者') {
    console.log(`ブログ記事の生成を開始: ${topic}`);
    
    // 1. リサーチフェーズ
    const research = await this.researchAgent.text(`
      「${topic}」について以下の観点から調査してください：
      1. 最新のトレンドと動向
      2. 実用的な使用例
      3. よくある課題と解決策
      4. ベストプラクティス
      
      対象読者: ${targetAudience}
    `);
    
    // 2. 執筆フェーズ
    const article = await this.writerAgent.text(`
      以下の調査結果を基に、魅力的な技術記事を作成してください：
      
      調査結果：
      ${research.text}
      
      記事の要件：
      - タイトルは興味を引くものに
      - 実際のコード例を含める
      - 段階的な説明を心がける
      - マークダウン形式で出力
      - 読者が実際に試せる内容にする
      
      対象読者: ${targetAudience}
    `);
    
    // 3. 編集・最適化フェーズ
    const finalArticle = await this.editorAgent.text(`
      以下の記事を編集し、品質向上とSEO最適化を行ってください：
      
      ${article.text}
      
      編集要件：
      - 文章の流れと読みやすさの改善
      - 適切な見出し構造の確保
      - SEOキーワードの自然な配置
      - コードサンプルの品質チェック
      - 最終的な品質確保
    `);
    
    // ファイルに保存
    const fileName = `${topic.replace(/[^a-zA-Z0-9]/g, '-')}-${Date.now()}.md`;
    await this.writerAgent.text(`
      以下の記事内容をファイル 'output/${fileName}' に保存してください：
      
      ${finalArticle.text}
    `);
    
    return {
      article: finalArticle.text,
      fileName,
      generatedAt: new Date().toISOString()
    };
  }
}

// 使用例
export async function generateTechBlog() {
  const generator = new BlogGenerator();
  
  const result = await generator.generateBlogPost(
    'Mastra フレームワークを使ったAI開発',
    'TypeScript開発者'
  );
  
  console.log('記事生成完了:', result.fileName);
  return result;
}
```

### 2. コードレビューアシスタント

```typescript:src/applications/code-reviewer.ts
import { Agent } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

class CodeReviewAssistant {
  private reviewerAgent: Agent;
  
  constructor() {
    this.reviewerAgent = new Agent({
      name: 'CodeReviewer',
      instructions: `
        あなたは経験豊富なシニア開発者として、コードレビューを行います。
        以下の観点でレビューしてください：
        1. コード品質（可読性、保守性）
        2. パフォーマンス
        3. セキュリティ
        4. ベストプラクティスの遵守
        5. 潜在的なバグ
        
        建設的なフィードバックと具体的な改善提案を提供してください。
      `,
      model: openai('gpt-4o-mini'),
    });
  }
  
  async reviewCode(code: string, language: string = 'typescript') {
    const review = await this.reviewerAgent.text(`
      以下の${language}コードをレビューしてください：
      
      \`\`\`${language}
      ${code}
      \`\`\`
      
      レビュー形式：
      ## 全体的な評価
      - 評価スコア: [1-10]
      - 主な問題点
      
      ## 詳細な指摘
      ### 良い点
      - [具体的な良い点]
      
      ### 改善点
      - [具体的な改善提案]
      
      ### セキュリティ
      - [セキュリティ関連の指摘]
      
      ## 改善されたコード例
      \`\`\`${language}
      [改善されたコード]
      \`\`\`
    `);
    
    return review.text;
  }
  
  async reviewPullRequest(diffContent: string) {
    const review = await this.reviewerAgent.text(`
      以下のGitディフを精査し、プルリクエストレビューを行ってください：
      
      ${diffContent}
      
      レビュー観点：
      1. 変更内容の妥当性
      2. 既存コードへの影響
      3. テストカバレッジ
      4. ドキュメントの更新が必要か
      5. 破壊的変更の有無
      
      GitHub形式のレビューコメントで回答してください。
    `);
    
    return review.text;
  }
}

export async function performCodeReview() {
  const reviewer = new CodeReviewAssistant();
  
  const sampleCode = `
  function fetchUserData(userId: string) {
    const user = fetch('/api/users/' + userId)
      .then(response => response.json())
      .then(data => {
        return data;
      })
      .catch(error => {
        console.log(error);
      });
    return user;
  }
  `;
  
  const review = await reviewer.reviewCode(sampleCode);
  console.log('コードレビュー結果:', review);
  
  return review;
}
```

## 本番環境での運用

### 1. エラーハンドリングと監視

```typescript:src/monitoring/error-handling.ts
import { Agent } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

class ResilientAgent {
  private agent: Agent;
  private retryCount: number = 0;
  private maxRetries: number = 3;
  
  constructor(config: any) {
    this.agent = new Agent(config);
  }
  
  async executeWithRetry(prompt: string, options?: any): Promise<string> {
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        const result = await this.agent.text(prompt, options);
        this.retryCount = 0; // 成功時はリセット
        return result.text;
        
      } catch (error) {
        console.error(`試行 ${attempt} 失敗:`, error);
        
        if (attempt === this.maxRetries) {
          throw new Error(`${this.maxRetries}回の試行後も失敗: ${error}`);
        }
        
        // 指数バックオフ
        const delay = Math.pow(2, attempt - 1) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
    
    throw new Error('予期しないエラー');
  }
  
  async healthCheck(): Promise<boolean> {
    try {
      await this.agent.text('Hello, test message');
      return true;
    } catch {
      return false;
    }
  }
}

// 使用例
export async function createResilientAgent() {
  const resilientAgent = new ResilientAgent({
    name: 'ProductionAgent',
    instructions: '本番環境で安定動作するエージェント',
    model: openai('gpt-4o-mini'),
  });
  
  // ヘルスチェック
  const isHealthy = await resilientAgent.healthCheck();
  console.log('エージェントの状態:', isHealthy ? '正常' : '異常');
  
  if (isHealthy) {
    const result = await resilientAgent.executeWithRetry(
      '簡単なテストメッセージに返答してください'
    );
    console.log('テスト結果:', result);
  }
  
  return resilientAgent;
}
```

### 2. パフォーマンス最適化

```typescript:src/optimization/performance.ts
import { Agent, Cache } from '@mastra/core';
import { openai } from '@mastra/provider-openai';

// キャッシュ設定
const cache = new Cache({
  type: 'redis',
  url: process.env.REDIS_URL,
  ttl: 3600 // 1時間
});

class OptimizedAgent {
  private agent: Agent;
  private cache: Cache;
  
  constructor() {
    this.agent = new Agent({
      name: 'OptimizedAgent',
      instructions: 'パフォーマンス最適化されたエージェント',
      model: openai('gpt-4o-mini'),
    });
    
    this.cache = cache;
  }
  
  async executeWithCache(prompt: string, cacheKey?: string): Promise<string> {
    const key = cacheKey || this.generateCacheKey(prompt);
    
    // キャッシュから取得を試行
    const cached = await this.cache.get(key);
    if (cached) {
      console.log('キャッシュから結果を返却');
      return cached;
    }
    
    // キャッシュにない場合は実行
    const result = await this.agent.text(prompt);
    
    // 結果をキャッシュに保存
    await this.cache.set(key, result.text);
    
    return result.text;
  }
  
  private generateCacheKey(prompt: string): string {
    // プロンプトのハッシュ値をキーとして使用
    const crypto = require('crypto');
    return crypto.createHash('md5').update(prompt).digest('hex');
  }
  
  async batchProcess(prompts: string[]): Promise<string[]> {
    // 並列処理で複数のプロンプトを効率的に処理
    const promises = prompts.map(prompt => 
      this.executeWithCache(prompt)
    );
    
    return Promise.all(promises);
  }
}

export async function demonstrateOptimization() {
  const optimizedAgent = new OptimizedAgent();
  
  // 複数のタスクを並列処理
  const prompts = [
    'TypeScriptの利点を3つ教えて',
    'Reactのベストプラクティスをまとめて',
    'Node.jsでのエラーハンドリング方法'
  ];
  
  console.time('バッチ処理時間');
  const results = await optimizedAgent.batchProcess(prompts);
  console.timeEnd('バッチ処理時間');
  
  console.log('処理結果:', results.length, '件');
  
  return results;
}
```

## まとめ

本記事では、**MastraフレームワークでTypeScriptプロジェクトにAIエージェントを組み込む方法**を包括的に解説しました。

**🎯 Mastraの主要な利点**

>* **TypeScriptネイティブ**: 型安全性と開発体験の向上
>* **プロダクション対応**: エラーハンドリングと監視機能が充実
>* **柔軟なアーキテクチャ**: エージェント、ワークフロー、RAGの組み合わせ
>* **豊富な統合**: 外部サービスとの連携が容易

**💡 実装のポイント**

>* **段階的導入**: 基本的なエージェントから始めて機能を拡張
>* **カスタムツール**: 業務に特化したツールで差別化
>* **ワークフロー活用**: 複雑なタスクを自動化
>* **RAG統合**: 専門知識を活用した高精度な回答

**⚡ 次のステップ**

>* **プロダクション環境への展開**: 監視とエラーハンドリングの強化
>* **スケールアップ**: 複数エージェントの協調システム構築
>* **カスタマイゼーション**: 組織固有のニーズに合わせた最適化

Mastraは、AIエージェント開発の複雑さを抽象化しながら、実用的で拡張性の高いソリューションを提供します。ぜひあなたのプロジェクトでも活用してみてください！