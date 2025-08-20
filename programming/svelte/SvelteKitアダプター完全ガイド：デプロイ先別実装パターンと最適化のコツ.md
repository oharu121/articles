# SvelteKitアダプター完全ガイド：デプロイ先別実装パターンと最適化のコツ

## はじめに

SvelteKitでアプリを作ったはいいけど、デプロイ先によってアダプターを切り替えなくちゃいけないとか、設定がややこしくて困ってませんか？

「Vercel用にビルドしたつもりが、Node.jsサーバーでエラーが出る」「静的サイトとして出力したいのに、SSRが動いてしまう」なんて経験はありませんか？

この記事では、**SvelteKitの各種アダプターの実装パターンと使い分けのコツ**を実際のコード例とともに解説します。デプロイ先に応じた最適なアダプターの選び方から、パフォーマンス最適化の実装まで、実務で即使える内容をお届けします！

## SvelteKitアダプターとは何か

SvelteKitのアダプター（Adapter）は、**ビルド時にアプリケーションをデプロイ先の環境に最適化する仕組み**です。

```typescript:svelte.config.js
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/kit/vite';

const config = {
  kit: {
    adapter: adapter()
  },
  preprocess: vitePreprocess()
};

export default config;
```

> 💡 **アダプターが解決する問題**
> * デプロイ先ごとに異なるランタイム要件への対応
> * SSR、SSG、SPAの出力形式の切り替え
> * 環境に応じた最適化とバンドリング

## 主要アダプター一覧と使い分け

| アダプター | 用途 | 出力形式 | SSR |
|------------|------|----------|-----|
| `adapter-auto` | 自動検出（推奨） | 環境依存 | ✅ |
| `adapter-vercel` | Vercel専用 | Edge/Serverless | ✅ |
| `adapter-netlify` | Netlify専用 | Edge/Functions | ✅ |
| `adapter-cloudflare` | Cloudflare Pages | Edge Workers | ✅ |
| `adapter-node` | Node.jsサーバー | Express風 | ✅ |
| `adapter-static` | 静的サイト | HTML/CSS/JS | ❌ |

## デプロイ先別実装パターン

### Vercel向けの実装

```bash
npm i -D @sveltejs/adapter-vercel
```

```typescript:svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

const config = {
  kit: {
    adapter: adapter({
      // Edge Runtime を使用（推奨）
      runtime: 'edge',
      // リージョン指定
      regions: ['nrt1'],
      // 分析データ送信
      analytics: true
    })
  }
};

export default config;
```

**Edge Functionsを活用したAPIルート:**

```typescript:src/routes/api/user/+server.ts
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ url, platform }) => {
  // Vercel Edge Runtime での地理的情報取得
  const country = platform?.env?.VERCEL_REGION ?? 'unknown';
  
  const response = {
    message: 'Hello from Edge!',
    region: country,
    timestamp: new Date().toISOString()
  };
  
  return new Response(JSON.stringify(response), {
    headers: {
      'content-type': 'application/json',
      'cache-control': 's-maxage=60'
    }
  });
};
```

### Node.js環境向けの実装

```bash
npm i -D @sveltejs/adapter-node
```

```typescript:svelte.config.js
import adapter from '@sveltejs/adapter-node';

const config = {
  kit: {
    adapter: adapter({
      // 出力ディレクトリ
      out: 'build',
      // 圧縮を有効化
      compress: true,
      // プリレンダリング
      precompress: true,
      // 環境変数のプレフィックス
      envPrefix: 'MY_APP_'
    })
  }
};

export default config;
```

**カスタムサーバーとの統合:**

```typescript:server.js
import { handler } from './build/handler.js';
import express from 'express';
import compression from 'compression';

const app = express();

// ミドルウェア設定
app.use(compression());
app.use('/assets', express.static('build/client/assets'));

// SvelteKitハンドラー統合
app.use(handler);

// ヘルスチェックエンドポイント
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: Date.now() });
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Server running on http://localhost:${port}`);
});
```

### 静的サイト生成の実装

```bash
npm i -D @sveltejs/adapter-static
```

```typescript:svelte.config.js
import adapter from '@sveltejs/adapter-static';

const config = {
  kit: {
    adapter: adapter({
      // 出力ディレクトリ
      pages: 'build',
      // アセットディレクトリ
      assets: 'build',
      // フォールバックページ
      fallback: '404.html',
      // プリレンダリング設定
      precompress: true
    }),
    // すべてのページをプリレンダリング
    prerender: {
      default: true,
      entries: ['*']
    }
  }
};

export default config;
```

**プリレンダリング制御:**

```typescript:src/routes/blog/[slug]/+page.ts
import type { PageLoad } from './$types';

export const prerender = true;

export const load: PageLoad = async ({ params, fetch }) => {
  const response = await fetch(`/api/posts/${params.slug}`);
  const post = await response.json();
  
  return {
    post
  };
};
```

## パフォーマンス最適化の実装

### チャンク分割とコード分離

```typescript:vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [sveltekit()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // 外部ライブラリを分離
          vendor: ['svelte', '@sveltejs/kit'],
          // UI関連を分離  
          ui: ['lucide-svelte', '@floating-ui/dom'],
          // 大きなライブラリを分離
          charts: ['chart.js', 'd3']
        }
      }
    }
  },
  // 開発時のオプティマイゼーション
  optimizeDeps: {
    include: ['chart.js', 'd3']
  }
});
```

### 画像最適化アダプター

```typescript:src/lib/imageOptimizer.ts
import type { Handle } from '@sveltejs/kit';
import sharp from 'sharp';

export const imageOptimizationHandle: Handle = async ({ event, resolve }) => {
  const { url, request } = event;
  
  // 画像リクエストの検出
  if (url.pathname.startsWith('/images/') && url.searchParams.has('w')) {
    const imagePath = `static${url.pathname}`;
    const width = parseInt(url.searchParams.get('w') || '800');
    const quality = parseInt(url.searchParams.get('q') || '80');
    
    try {
      const optimizedImage = await sharp(imagePath)
        .resize(width)
        .jpeg({ quality })
        .toBuffer();
        
      return new Response(optimizedImage, {
        headers: {
          'Content-Type': 'image/jpeg',
          'Cache-Control': 'public, max-age=31536000'
        }
      });
    } catch (error) {
      // フォールバックとして元の画像を返す
      return resolve(event);
    }
  }
  
  return resolve(event);
};
```

### 環境別設定管理

```typescript:src/lib/config.ts
interface Config {
  apiUrl: string;
  analyticsId?: string;
  features: {
    enableAnalytics: boolean;
    enablePWA: boolean;
  };
}

const configs: Record<string, Config> = {
  development: {
    apiUrl: 'http://localhost:5173/api',
    features: {
      enableAnalytics: false,
      enablePWA: false
    }
  },
  
  production: {
    apiUrl: 'https://api.example.com',
    analyticsId: 'G-XXXXXXXXXX',
    features: {
      enableAnalytics: true,
      enablePWA: true
    }
  }
};

export const config = configs[import.meta.env.MODE] || configs.development;
```

## アダプター選択のベストプラクティス

### 決定フローチャート

```
デプロイ先は？
├── Vercel → adapter-vercel
├── Netlify → adapter-netlify  
├── Cloudflare → adapter-cloudflare
├── 自前サーバー
│   ├── Node.js → adapter-node
│   └── 静的ファイルのみ → adapter-static
└── わからない → adapter-auto（推奨）
```

### アダプター別トラブルシューティング

> 🔍 **よくある問題と対策**

**adapter-static でSSRエラー**
```typescript
// ❌ 動的な処理はNG
export const load = async ({ fetch }) => {
  const response = await fetch('/api/dynamic');
  return await response.json();
};

// ✅ ビルド時に解決
export const prerender = true;
export const load = async ({ fetch }) => {
  const response = await fetch('https://api.example.com/static');
  return await response.json();
};
```

**adapter-vercel でEdgeランタイムエラー**
```typescript
// ❌ Node.js APIは使用不可
import fs from 'fs';

// ✅ Web標準APIを使用
const response = await fetch('https://api.example.com/data');
```

## まとめ

SvelteKitのアダプターは、デプロイ先に応じて最適化されたアプリケーションを生成する強力な仕組みです。

> ✅ **実装のポイント**
> 
> * `adapter-auto`で始めて、必要に応じて専用アダプターに切り替える
> * 各アダプターの特性（Edge Runtime、Node.js、静的生成）を理解する
> * 環境に応じたコード分割と最適化を実装する
> * パフォーマンス要件に合わせてアダプターを選択する

これで、SvelteKitアプリを任意の環境で効率よくデプロイできるようになりましたね！次のプロジェクトでは、ぜひこの知識を活用して、最適なアダプター選択とパフォーマンス最適化に挑戦してみてください。