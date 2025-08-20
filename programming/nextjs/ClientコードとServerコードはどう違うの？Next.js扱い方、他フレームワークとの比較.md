## Client コードと Server コードはどう違うの？Next.js 扱い方、他フレームワークとの比較

## はじめに

Next.js（App Router）を使い始めると、まずぶつかるのが **Server Component** と **Client Component** の違いです。

「`useState`が使えない？」「`window`がエラーになる？」
そう、React はもはや **“1 つの実行環境”ではなくなった** のです。

## 「別々の環境」と考えると理解が早い

この違いを理解するコツは、**Client と Server は全く別の実行環境**だと割り切ることです。

**🖥️ Client = ブラウザ**

- ユーザーのブラウザで動く
- `window` や `document` が使える
- UI の操作、クリックイベント、DOM の取得などが可能
- サーバーから送られてきた JavaScript が**ブラウザ上で実行される**

**🌐 Server = Node.js 上で実行される**

- ブラウザ API（`window`, `localStorage` など）は存在しない
- データベースアクセス、認証トークンの処理などが可能
- HTML を生成して**クライアントに送る役割**を持つ

**Server Component と Client Component の違い**

|                    | Server Component               | Client Component            |
| ------------------ | ------------------------------ | --------------------------- |
| 実行環境           | サーバー（Node.js）            | クライアント（ブラウザ）    |
| `'use client'`     | ❌ 不要（デフォルト）          | ✅ 必須                     |
| React フック       | ❌ `useState`, `useEffect`不可 | ✅ 使える                   |
| DOM 操作           | ❌ 不可                        | ✅ OK                       |
| DB/API アクセス    | ✅ 得意                        | △ 可能だが非効率            |
| Nodejs ライブラリ  | ✅ 得意                        | ❌ 不可（バンドル対象外）   |
| ファイルの読み込み | ✅ 可能（`fs`など）            | ❌ 不可（ブラウザでは不可） |

## Next.js での使い方

**🌐 Server Component（デフォルト）**

```tsx:app/page.tsx
// デフォルトでServer Component
import { getPosts } from '@/lib/api';

export default async function Page() {
  const posts = await getPosts();
  return <PostList posts={posts} />;
}
```

> - DB から記事一覧を取得して静的なリストとして返す用途に向く
> - 実行はサーバー上（Node.js）
> - React の `renderToStream` などで **HTML を直接生成**
> - JavaScript は基本的にクライアントに送られない（＝**クライアントは再実行しない**）
>   ➡ クライアントには **レンダリング済みの HTML** だけが届く

**🖥️ Client Component**

```tsx:components/LikeButton.tsx
'use client';

import { useState } from 'react';

export default function LikeButton() {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? '❤️' : '🤍'}</button>;
}
```

> - 用途：`useState`, `onClick` を使いたいとき
> - `use client` を付けると、JS としてクライアントにバンドルされる
> - 初期表示は Server から HTML が送られる（**SSR + Hydration**）
> - 以降はクライアントで状態管理、DOM 更新、イベント処理が行われる
>   ➡ JavaScript バンドルがブラウザに送られ、ブラウザで React が HTML を構築

**✨ 一文で言うなら**

> Server Component は **HTML を「出力する」コンポーネント**、Client Component は **ブラウザで「動作する」コンポーネント**

## よくある失敗とその対策

#### 1. 値のズレによる Hydration Error

> Server と Client は環境が違うだけでなく、**タイミングも違う**ため、同じ結果を出すとは限りません。

```tsx
export default function Time() {
  return <p>{Date.now()}</p>;
}
```

Server と Client で `Date.now()` の値がズレる → Hydration Error 発生

```bash
Warning: Text content did not match. Server: "1650000000000" Client: "1650000002000"
```

✅ 安全な実装

```tsx
"use client";

import { useEffect, useState } from "react";

export default function Time() {
  const [now, setNow] = useState<number | null>(null);

  useEffect(() => {
    setNow(Date.now());
  }, []);

  return <p>{now ?? "Loading..."}</p>;
}
```

#### 2. `window` を server で使用する

```tsx
if (!window) return; // ReferenceError: window is not defined
```

✅ 正しい使い方

```tsx
if (typeof window === "undefined") return; // サーバーでも安全
```

とはいえ、ベストなのは**Client Component + useEffect 内で使うこと**：

```tsx
"use client";

useEffect(() => {
  if (typeof window !== "undefined") {
    console.log(window.innerWidth);
  }
}, []);
```

## 他フレームワークでの扱い方

#### 1. Nuxt 3

```vue
<template>
  <p v-if="width">Width: {{ width }}px</p>
</template>

<script setup>
import { ref, onMounted } from "vue";

const width = ref(null);

onMounted(() => {
  width.value = window.innerWidth;
});
</script>
```

> `onMounted` はブラウザ環境でのみ呼ばれるため安全

#### 2. SvelteKit

```svelte
<script>
  import { onMount } from 'svelte';
  let width = 0;

  onMount(() => {
    width = window.innerWidth;
  });
</script>

{#if width}
  <p>Width: {width}px</p>
{/if}
```

> `onMount()` は SvelteKit で「クライアントのみ実行」される構文

#### 3. Remix

Remix は基本的に React と同じですが、`useEffect`の中でクライアント専用処理を行うのが安全です。

```tsx
import { useEffect, useState } from "react";

export default function WidthViewer() {
  const [width, setWidth] = useState(0);

  useEffect(() => {
    setWidth(window.innerWidth);
  }, []);

  return <p>Width: {width}px</p>;
}
```

> Remix には Server/Client の明示的な分離はなく、Client コードは自己責任で書く必要がある

#### 4. Angular

> - Angular は基本的にすべて**クライアント側で動く SPA**
> - **DI（依存性注入）** によってクライアント/サーバーの分岐を柔軟に制御
> - SSR でも、**特定の処理を「クライアント限定」にする必要がある**

Angular で`window`や`document`をそのまま使うと、SSR 時にエラーになります。

**❌ NG な例（SSR でクラッシュ）**

```ts
ngOnInit() {
  console.log(window.innerWidth); // ❌ SSRではReferenceError
}
```

**✅ 安全な方法：`isPlatformBrowser()`で判定**

```ts
import { isPlatformBrowser } from '@angular/common';
import { PLATFORM_ID, Inject } from '@angular/core';

constructor(@Inject(PLATFORM_ID) private platformId: Object) {}

ngOnInit() {
  if (isPlatformBrowser(this.platformId)) {
    console.log(window.innerWidth); // ✅ クライアントのみ
  }
}
```

**各フレームワークの比較まとめ**

| フレームワーク  | Server / Client 分離の設計   | クライアント限定コードの扱い方       |
| --------------- | ---------------------------- | ------------------------------------ |
| **Next.js**     | 明示的（`'use client'`）     | Client Component 内で `useEffect` 等 |
| **Nuxt3 (Vue)** | 条件分岐（`process.client`） | `<client-only>` など                 |
| **SvelteKit**   | 明示的（`onMount`）          | `onMount()` 内に書く                 |
| **Remix**       | 分離なし                     | `useEffect` 等で制御                 |
| **Angular**     | 分離なし                     | `isPlatformBrowser()` で明示分岐     |

## おわりに

Next.js の Server Component と Client Component は、環境の違いを意識しながら設計することで、より安全で効率的なアプリ開発ができるようになります。

特に「**Client と Server は別の環境**」という視点を持つだけで、

- なぜ`window`が使えないのか
- なぜ`use client`が必要なのか
- なぜ Hydration Error が起こるのか

といった疑問の多くが、シンプルに整理できるようになります。

また、今回紹介した他フレームワークとの比較からも、Next.js が Server Component を中心に据えたことで、**構文的に安全なアーキテクチャを築いている**ことがわかります。
