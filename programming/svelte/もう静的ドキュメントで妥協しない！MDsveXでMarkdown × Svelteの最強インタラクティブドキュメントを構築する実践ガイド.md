# もう静的ドキュメントで妥協しない！MDsveXでMarkdown × Svelteの最強インタラクティブドキュメントを構築する実践ガイド

## はじめに

技術ドキュメントやブログを書いている時、「静的なMarkdownじゃ物足りない...もっとインタラクティブに説明したい！」って思ったことはありませんか？

コードサンプルを実際に動かして見せたい、グラフやチャートを埋め込みたい、フォームの動作を実演したい...でも、通常のMarkdownでは静的なテキストと画像だけで終わってしまいますよね。

かといって、HTMLやReactでゴリゴリ書くのは記事執筆の手軽さが失われてしまうし、コンテンツとコードが混在して保守性も悪い。

この記事では、**MDsveXでMarkdownの手軽さとSvelteのインタラクティビティを融合した最強ドキュメント作成手法**を実装例とともに徹底解説します！ブログ、技術ドキュメント、プレゼン資料まで、あらゆるコンテンツを次のレベルに押し上げる実践的な内容をお届けします。

## MDsveXとは何か

MDsveX（エム・ディー・スベックス）は、**MarkdownとSvelteを融合したプリプロセッサ**です。

**💡 MDsveXの核心**

> * MarkdownファイルにSvelteコンポーネントを直接埋め込める
> * 通常のMarkdown記法はそのまま使用可能
> * SvelteKitと完全統合されたスムーズな開発体験

```svelte:example.svx
# 通常のMarkdownとして書ける

ここに**普通の**Markdownコンテンツを書けます。

<script>
  let count = 0;
</script>

<!-- そして、Svelteコンポーネントも直接使える！ -->
<button on:click={() => count++}>
  クリック数: {count}
</button>
```

## 基本的な実装セットアップ

### プロジェクトの初期化

```bash
npm create svelte@latest mdsvex-demo
cd mdsvex-demo
npm install
```

### MDsveXの導入

```bash
npm install -D mdsvex
```

```typescript:svelte.config.js
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/kit/vite';
import { mdsvex } from 'mdsvex';

const config = {
  kit: {
    adapter: adapter()
  },
  
  extensions: ['.svelte', '.svx', '.md'],
  
  preprocess: [
    vitePreprocess(),
    mdsvex({
      extensions: ['.svx', '.md'],
      layout: {
        // レイアウトファイルを指定
        _: './src/lib/MdLayout.svelte'
      }
    })
  ]
};

export default config;
```

### ベースレイアウトの作成

```svelte:src/lib/MdLayout.svelte
<script>
  // 各.svxファイルのメタデータを受け取る
  export let title = '';
  export let description = '';
  export let date = '';
  export let tags = [];
</script>

<svelte:head>
  <title>{title}</title>
  <meta name="description" content={description} />
</svelte:head>

<article class="prose max-w-4xl mx-auto px-6 py-8">
  <!-- メタ情報表示 -->
  {#if date}
    <div class="article-meta">
      <time class="text-gray-500">{date}</time>
      {#if tags.length > 0}
        <div class="tags">
          {#each tags as tag}
            <span class="tag">#{tag}</span>
          {/each}
        </div>
      {/if}
    </div>
  {/if}

  <!-- Markdownコンテンツがここにレンダリングされる -->
  <slot />
</article>

<style>
  .article-meta {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
    padding-bottom: 1rem;
    border-bottom: 1px solid #e5e7eb;
  }
  
  .tags {
    display: flex;
    gap: 0.5rem;
  }
  
  .tag {
    background: #f3f4f6;
    color: #374151;
    padding: 0.25rem 0.5rem;
    border-radius: 0.375rem;
    font-size: 0.875rem;
  }
</style>
```

## インタラクティブコンポーネントの実装

### カウンターコンポーネント

```svelte:src/lib/components/Counter.svelte
<script lang="ts">
  export let initial: number = 0;
  export let step: number = 1;
  export let max: number | null = null;
  export let min: number | null = null;
  
  let count = initial;
  
  function increment() {
    if (max === null || count + step <= max) {
      count += step;
    }
  }
  
  function decrement() {
    if (min === null || count - step >= min) {
      count -= step;
    }
  }
  
  function reset() {
    count = initial;
  }
</script>

<div class="counter">
  <div class="display">
    <span class="count">{count}</span>
    {#if max !== null && count === max}
      <span class="status max">MAX!</span>
    {:else if min !== null && count === min}
      <span class="status min">MIN!</span>
    {/if}
  </div>
  
  <div class="controls">
    <button 
      on:click={decrement}
      disabled={min !== null && count <= min}
    >
      -
    </button>
    
    <button on:click={reset}>
      リセット
    </button>
    
    <button 
      on:click={increment}
      disabled={max !== null && count >= max}
    >
      +
    </button>
  </div>
</div>

<style>
  .counter {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1rem;
    padding: 1.5rem;
    border: 2px solid #e5e7eb;
    border-radius: 0.5rem;
    background: #f9fafb;
    margin: 1rem 0;
  }
  
  .display {
    display: flex;
    align-items: center;
    gap: 1rem;
  }
  
  .count {
    font-size: 2rem;
    font-weight: bold;
    color: #1f2937;
    min-width: 3rem;
    text-align: center;
  }
  
  .status {
    font-size: 0.875rem;
    font-weight: bold;
    padding: 0.25rem 0.5rem;
    border-radius: 0.25rem;
  }
  
  .status.max {
    background: #fef3c7;
    color: #92400e;
  }
  
  .status.min {
    background: #fee2e2;
    color: #991b1b;
  }
  
  .controls {
    display: flex;
    gap: 0.5rem;
  }
  
  button {
    padding: 0.5rem 1rem;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem;
    background: white;
    cursor: pointer;
    font-weight: 500;
    transition: all 0.2s;
  }
  
  button:hover:not(:disabled) {
    background: #f3f4f6;
    border-color: #9ca3af;
  }
  
  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
```

### APIデモコンポーネント

```svelte:src/lib/components/ApiDemo.svelte
<script lang="ts">
  export let endpoint: string = '/api/demo';
  export let method: 'GET' | 'POST' | 'PUT' | 'DELETE' = 'GET';
  export let initialData: Record<string, any> = {};
  
  let requestData = JSON.stringify(initialData, null, 2);
  let response = '';
  let loading = false;
  let error = '';
  
  async function sendRequest() {
    loading = true;
    error = '';
    response = '';
    
    try {
      const options: RequestInit = {
        method,
        headers: {
          'Content-Type': 'application/json',
        }
      };
      
      if (method !== 'GET' && requestData.trim()) {
        options.body = requestData;
      }
      
      const res = await fetch(endpoint, options);
      const data = await res.json();
      
      response = JSON.stringify(data, null, 2);
      
      if (!res.ok) {
        error = `HTTP ${res.status}: ${res.statusText}`;
      }
    } catch (err) {
      error = err instanceof Error ? err.message : 'Unknown error';
    } finally {
      loading = false;
    }
  }
</script>

<div class="api-demo">
  <div class="endpoint">
    <span class="method {method.toLowerCase()}">{method}</span>
    <span class="url">{endpoint}</span>
    <button on:click={sendRequest} disabled={loading}>
      {loading ? '送信中...' : '実行'}
    </button>
  </div>
  
  {#if method !== 'GET'}
    <div class="request">
      <label for="request-data">リクエストボディ:</label>
      <textarea 
        id="request-data"
        bind:value={requestData}
        placeholder="JSONデータを入力..."
        rows="6"
      ></textarea>
    </div>
  {/if}
  
  {#if error}
    <div class="error">
      <strong>エラー:</strong> {error}
    </div>
  {/if}
  
  {#if response}
    <div class="response">
      <label>レスポンス:</label>
      <pre>{response}</pre>
    </div>
  {/if}
</div>

<style>
  .api-demo {
    border: 1px solid #d1d5db;
    border-radius: 0.5rem;
    padding: 1rem;
    margin: 1rem 0;
    background: #ffffff;
  }
  
  .endpoint {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-bottom: 1rem;
  }
  
  .method {
    padding: 0.25rem 0.75rem;
    border-radius: 0.25rem;
    font-weight: bold;
    font-size: 0.875rem;
    color: white;
  }
  
  .method.get { background: #10b981; }
  .method.post { background: #f59e0b; }
  .method.put { background: #3b82f6; }
  .method.delete { background: #ef4444; }
  
  .url {
    font-family: monospace;
    background: #f3f4f6;
    padding: 0.25rem 0.5rem;
    border-radius: 0.25rem;
    flex: 1;
  }
  
  .request textarea {
    width: 100%;
    font-family: monospace;
    border: 1px solid #d1d5db;
    border-radius: 0.375rem;
    padding: 0.5rem;
  }
  
  .error {
    background: #fef2f2;
    color: #dc2626;
    padding: 0.75rem;
    border-radius: 0.375rem;
    border-left: 4px solid #dc2626;
  }
  
  .response pre {
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 0.375rem;
    padding: 1rem;
    overflow-x: auto;
    font-size: 0.875rem;
  }
  
  button {
    padding: 0.5rem 1rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 0.375rem;
    cursor: pointer;
    font-weight: 500;
  }
  
  button:hover:not(:disabled) {
    background: #2563eb;
  }
  
  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
```

## TypeScriptでの高度な活用例

### データ可視化コンポーネント

```svelte:src/lib/components/DataChart.svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { Chart, registerables } from 'chart.js';
  
  Chart.register(...registerables);
  
  export interface ChartData {
    labels: string[];
    datasets: {
      label: string;
      data: number[];
      backgroundColor?: string;
      borderColor?: string;
    }[];
  }
  
  export let data: ChartData;
  export let type: 'line' | 'bar' | 'pie' | 'doughnut' = 'line';
  export let height: number = 300;
  export let title: string = '';
  
  let canvas: HTMLCanvasElement;
  let chart: Chart | null = null;
  
  onMount(() => {
    if (canvas) {
      chart = new Chart(canvas, {
        type,
        data,
        options: {
          responsive: true,
          maintainAspectRatio: false,
          plugins: {
            title: {
              display: !!title,
              text: title
            }
          }
        }
      });
    }
    
    return () => {
      if (chart) {
        chart.destroy();
      }
    };
  });
  
  // データ更新時にチャートを更新
  $: if (chart && data) {
    chart.data = data;
    chart.update();
  }
</script>

<div class="chart-container" style="height: {height}px;">
  <canvas bind:this={canvas}></canvas>
</div>

<style>
  .chart-container {
    margin: 1rem 0;
    position: relative;
  }
</style>
```

### インタラクティブコードエディタ

```svelte:src/lib/components/CodeRunner.svelte
<script lang="ts">
  import { onMount } from 'svelte';
  
  export let initialCode: string = '';
  export let language: string = 'javascript';
  export let editable: boolean = true;
  
  let code = initialCode;
  let output = '';
  let error = '';
  let running = false;
  
  async function runCode() {
    if (!code.trim()) return;
    
    running = true;
    output = '';
    error = '';
    
    try {
      // セキュアなコード実行環境をシミュレート
      const result = await executeInSandbox(code);
      output = String(result);
    } catch (err) {
      error = err instanceof Error ? err.message : 'Execution error';
    } finally {
      running = false;
    }
  }
  
  // 実際の実装では、Web Workersやサンドボックス環境を使用
  async function executeInSandbox(code: string): Promise<any> {
    // 警告: 実際のプロダクションではevalの使用は避けてください
    const asyncFunction = new Function('return (async () => { ' + code + ' })()');
    return await asyncFunction();
  }
  
  function formatCode() {
    // 簡単なコードフォーマット
    code = code.replace(/;/g, ';\n').replace(/\{/g, ' {\n  ').replace(/\}/g, '\n}');
  }
</script>

<div class="code-runner">
  <div class="editor">
    <div class="toolbar">
      <span class="language">{language}</span>
      <div class="actions">
        {#if editable}
          <button on:click={formatCode} class="format">フォーマット</button>
        {/if}
        <button on:click={runCode} disabled={running} class="run">
          {running ? '実行中...' : '実行'}
        </button>
      </div>
    </div>
    
    <textarea 
      bind:value={code}
      readonly={!editable}
      spellcheck="false"
      rows="10"
    ></textarea>
  </div>
  
  {#if output || error}
    <div class="output">
      <div class="output-header">実行結果</div>
      {#if error}
        <pre class="error">{error}</pre>
      {:else}
        <pre class="success">{output}</pre>
      {/if}
    </div>
  {/if}
</div>

<style>
  .code-runner {
    border: 1px solid #e5e7eb;
    border-radius: 0.5rem;
    overflow: hidden;
    margin: 1rem 0;
  }
  
  .toolbar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0.5rem 1rem;
    background: #f9fafb;
    border-bottom: 1px solid #e5e7eb;
  }
  
  .language {
    font-family: monospace;
    font-size: 0.875rem;
    color: #6b7280;
  }
  
  .actions {
    display: flex;
    gap: 0.5rem;
  }
  
  .actions button {
    padding: 0.25rem 0.75rem;
    border: 1px solid #d1d5db;
    border-radius: 0.25rem;
    background: white;
    cursor: pointer;
    font-size: 0.875rem;
  }
  
  .run {
    background: #10b981 !important;
    color: white !important;
    border-color: #10b981 !important;
  }
  
  textarea {
    width: 100%;
    border: none;
    outline: none;
    font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
    font-size: 0.875rem;
    line-height: 1.5;
    padding: 1rem;
    background: #ffffff;
    resize: vertical;
  }
  
  .output {
    border-top: 1px solid #e5e7eb;
  }
  
  .output-header {
    padding: 0.5rem 1rem;
    background: #f3f4f6;
    font-weight: 500;
    font-size: 0.875rem;
    color: #374151;
  }
  
  .output pre {
    margin: 0;
    padding: 1rem;
    font-family: monospace;
    font-size: 0.875rem;
    white-space: pre-wrap;
  }
  
  .output pre.success {
    background: #f0fdf4;
    color: #166534;
  }
  
  .output pre.error {
    background: #fef2f2;
    color: #dc2626;
  }
</style>
```

### 実際のドキュメントファイル例

```svx:src/routes/docs/interactive-demo.svx
---
title: "インタラクティブな技術デモ"
description: "MDsveXで作るリッチなドキュメント"
date: "2024-01-20"
tags: ["mdsvex", "svelte", "typescript"]
---

<script>
  import Counter from '$lib/components/Counter.svelte';
  import ApiDemo from '$lib/components/ApiDemo.svelte';
  import DataChart from '$lib/components/DataChart.svelte';
  import CodeRunner from '$lib/components/CodeRunner.svelte';
  
  const chartData = {
    labels: ['1月', '2月', '3月', '4月', '5月'],
    datasets: [{
      label: '売上',
      data: [120, 190, 300, 500, 200],
      backgroundColor: '#3b82f6',
      borderColor: '#2563eb'
    }]
  };
  
  const sampleCode = `
console.log('Hello, MDsveX!');

const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(n => n * 2);

console.log('元の配列:', numbers);
console.log('2倍の配列:', doubled);

return doubled.reduce((a, b) => a + b, 0);
  `.trim();
</script>

# インタラクティブな技術デモ

## カウンターコンポーネント

通常のMarkdownでは表現できない、インタラクティブな要素を簡単に埋め込めます。

<Counter initial={0} step={2} max={10} min={-5} />

パラメータを変更して、異なる動作をテストできます：

<Counter initial={100} step={25} max={200} />

## APIデモンストレーション

実際のAPI呼び出しを体験できます：

<ApiDemo endpoint="/api/users" method="GET" />

POST APIのテスト：

<ApiDemo 
  endpoint="/api/users" 
  method="POST" 
  initialData={{name: "田中太郎", email: "tanaka@example.com"}} 
/>

## データ可視化

リアルタイムでデータを表示：

<DataChart data={chartData} type="bar" title="月別売上データ" />

## コード実行環境

実際にコードを実行してみましょう：

<CodeRunner 
  initialCode={sampleCode}
  language="javascript" 
/>

## 通常のMarkdownも使える

もちろん、**通常のMarkdown記法**も完全にサポートされています。

- リスト項目1
- リスト項目2
- リスト項目3

| 機能 | 通常のMarkdown | MDsveX |
|------|----------------|--------|
| テキスト | ✅ | ✅ |
| 画像 | ✅ | ✅ |
| インタラクティブ要素 | ❌ | ✅ |
| データバインディング | ❌ | ✅ |

```javascript
// コードブロックも当然使用可能
function greet(name) {
  return `Hello, ${name}!`;
}
```

この柔軟性こそが、MDsveXの最大の魅力です！
```

## ルーティングとナビゲーション

```typescript:src/routes/docs/+layout.svelte
<script lang="ts">
  import { page } from '$app/stores';
  
  const docPages = [
    { path: '/docs/getting-started', title: 'はじめに' },
    { path: '/docs/interactive-demo', title: 'インタラクティブデモ' },
    { path: '/docs/components', title: 'コンポーネント活用' },
    { path: '/docs/advanced', title: '高度な技法' }
  ];
</script>

<div class="docs-layout">
  <nav class="sidebar">
    <h2>ドキュメント</h2>
    <ul>
      {#each docPages as doc}
        <li>
          <a 
            href={doc.path}
            class:active={$page.url.pathname === doc.path}
          >
            {doc.title}
          </a>
        </li>
      {/each}
    </ul>
  </nav>
  
  <main class="content">
    <slot />
  </main>
</div>

<style>
  .docs-layout {
    display: grid;
    grid-template-columns: 250px 1fr;
    gap: 2rem;
    max-width: 1200px;
    margin: 0 auto;
    padding: 2rem;
  }
  
  .sidebar {
    border-right: 1px solid #e5e7eb;
    padding-right: 2rem;
  }
  
  .sidebar h2 {
    margin-top: 0;
    color: #374151;
  }
  
  .sidebar ul {
    list-style: none;
    padding: 0;
  }
  
  .sidebar li {
    margin-bottom: 0.5rem;
  }
  
  .sidebar a {
    display: block;
    padding: 0.5rem;
    text-decoration: none;
    color: #6b7280;
    border-radius: 0.25rem;
    transition: all 0.2s;
  }
  
  .sidebar a:hover,
  .sidebar a.active {
    background: #f3f4f6;
    color: #374151;
  }
  
  .content {
    min-width: 0;
  }
</style>
```

## SEOとメタデータの最適化

```typescript:src/lib/utils/metadata.ts
export interface ArticleMetadata {
  title: string;
  description: string;
  date: string;
  tags: string[];
  author?: string;
  image?: string;
}

export function generateJsonLd(metadata: ArticleMetadata, url: string) {
  return {
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": metadata.title,
    "description": metadata.description,
    "datePublished": metadata.date,
    "author": {
      "@type": "Person",
      "name": metadata.author || "Anonymous"
    },
    "keywords": metadata.tags.join(", "),
    "url": url,
    ...(metadata.image && { "image": metadata.image })
  };
}
```

```svelte:src/lib/MdLayout.svelte
<script lang="ts">
  import { page } from '$app/stores';
  import { generateJsonLd, type ArticleMetadata } from '$lib/utils/metadata';
  
  export let title = '';
  export let description = '';
  export let date = '';
  export let tags: string[] = [];
  export let author = '';
  export let image = '';
  
  $: metadata = { title, description, date, tags, author, image } as ArticleMetadata;
  $: jsonLd = generateJsonLd(metadata, $page.url.href);
</script>

<svelte:head>
  <title>{title}</title>
  <meta name="description" content={description} />
  <meta name="keywords" content={tags.join(', ')} />
  
  <!-- Open Graph -->
  <meta property="og:title" content={title} />
  <meta property="og:description" content={description} />
  <meta property="og:type" content="article" />
  <meta property="og:url" content={$page.url.href} />
  {#if image}
    <meta property="og:image" content={image} />
  {/if}
  
  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:title" content={title} />
  <meta name="twitter:description" content={description} />
  {#if image}
    <meta name="twitter:image" content={image} />
  {/if}
  
  <!-- JSON-LD -->
  {@html `<script type="application/ld+json">${JSON.stringify(jsonLd)}</script>`}
</svelte:head>

<!-- 既存のレイアウトコンテンツ -->
<slot />
```

## ビルドと最適化の設定

```typescript:vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [sveltekit()],
  
  optimizeDeps: {
    include: ['chart.js']
  },
  
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // MDsveX関連を分離
          mdsvex: ['mdsvex'],
          // チャート関連を分離
          charts: ['chart.js'],
          // エディタ関連を分離
          editor: ['codemirror', '@codemirror/lang-javascript']
        }
      }
    }
  }
});
```

## まとめ

MDsveXは、Markdownの簡潔さとSvelteのパワフルさを融合した革新的なツールです。

> ✅ **MDsveXで実現できること**
> 
> * 📝 **手軽な記述**: 通常のMarkdown記法をそのまま使用
> * 🚀 **インタラクティビティ**: リアルタイムなUI要素の埋め込み
> * 🎯 **TypeScript対応**: 型安全なコンポーネント開発
> * ⚡ **高パフォーマンス**: SvelteKitの最適化による高速表示
> * 🔧 **拡張性**: カスタムコンポーネントで無限の可能性

従来の静的ドキュメントの限界を打ち破り、読者にとって真にエンゲージングなコンテンツを作成できるようになりましたね！

技術記事、プロダクトドキュメント、教育コンテンツなど、あらゆる場面でMDsveXの威力を発揮してください。次のドキュメントプロジェクトでは、ぜひこの手法を活用して、読者の心を掴む素晴らしいコンテンツを作り上げてみてください！