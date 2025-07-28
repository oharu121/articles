# ClientコードとServerコードはどう違うの？Next.js扱い方、他フレームワークとの比較

# はじめに

Next.js（App Router）を使い始めると、まずぶつかるのが **Server Component** と **Client Component** の違いです。

「`useState`が使えない？」「`window`がエラーになる？」
そう、Reactはもはや **“1つの実行環境”ではなくなった** のです。

# 「別々の環境」と考えると理解が早い

この違いを理解するコツは、**ClientとServerは全く別の実行環境**だと割り切ることです。

**🖥️ Client = ブラウザ**

* ユーザーのブラウザで動く
* `window` や `document` が使える
* UIの操作、クリックイベント、DOMの取得などが可能
* サーバーから送られてきたJavaScriptが**ブラウザ上で実行される**

**🌐 Server = Node.js上で実行される**

* ブラウザAPI（`window`, `localStorage` など）は存在しない
* データベースアクセス、認証トークンの処理などが可能
* HTMLを生成して**クライアントに送る役割**を持つ

**Server ComponentとClient Componentの違い**

|                | Server Component            | Client Component  |
| -------------- | --------------------------- | ----------------- |
| 実行環境           | サーバー（Node.js）               | クライアント（ブラウザ）      |
| `'use client'` | ❌ 不要（デフォルト）                 | ✅ 必須              |
| Reactフック       | ❌ `useState`, `useEffect`不可 | ✅ 使える             |
| DOM操作          | ❌ 不可                        | ✅ OK              |
| DB/APIアクセス     | ✅ 得意                        | △ 可能だが非効率 |
| Nodejsライブラリ   | ✅ 得意                        | ❌ 不可（バンドル対象外）  |
| ファイルの読み込み  | ✅ 可能（`fs`など）            | ❌ 不可（ブラウザでは不可）  |

# Next.jsでの使い方

**🌐 Server Component（デフォルト）**

```tsx:app/page.tsx
// デフォルトでServer Component
import { getPosts } from '@/lib/api';

export default async function Page() {
  const posts = await getPosts();
  return <PostList posts={posts} />;
}
```
>* DBから記事一覧を取得して静的なリストとして返す用途に向く
>* 実行はサーバー上（Node.js）
>* React の `renderToStream` などで **HTMLを直接生成**
>* JavaScriptは基本的にクライアントに送られない（＝**クライアントは再実行しない**）
➡ クライアントには **レンダリング済みのHTML** だけが届く

**🖥️ Client Component**

```tsx:components/LikeButton.tsx
'use client';

import { useState } from 'react';

export default function LikeButton() {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? '❤️' : '🤍'}</button>;
}
```

>* 用途：`useState`, `onClick` を使いたいとき
>* `use client` を付けると、JSとしてクライアントにバンドルされる
>* 初期表示はServerからHTMLが送られる（**SSR + Hydration**）
>* 以降はクライアントで状態管理、DOM更新、イベント処理が行われる
➡ JavaScriptバンドルがブラウザに送られ、ブラウザでReactがHTMLを構築

**✨ 一文で言うなら**

> Server Componentは **HTMLを「出力する」コンポーネント**、Client Componentは **ブラウザで「動作する」コンポーネント**

# よくある失敗とその対策

### 1. 値のズレによるHydration Error

> ServerとClientは環境が違うだけでなく、**タイミングも違う**ため、同じ結果を出すとは限りません。

```tsx
export default function Time() {
  return <p>{Date.now()}</p>;
}
```

ServerとClientで `Date.now()` の値がズレる → Hydration Error発生

```bash
Warning: Text content did not match. Server: "1650000000000" Client: "1650000002000"
```

✅ 安全な実装

```tsx
'use client';

import { useEffect, useState } from 'react';

export default function Time() {
  const [now, setNow] = useState<number | null>(null);

  useEffect(() => {
    setNow(Date.now());
  }, []);

  return <p>{now ?? 'Loading...'}</p>;
}
```

### 2. `window` をserverで使用する

```tsx
if (!window) return; // ReferenceError: window is not defined
```

✅ 正しい使い方

```tsx
if (typeof window === 'undefined') return; // サーバーでも安全
```

とはいえ、ベストなのは**Client Component + useEffect内で使うこと**：

```tsx
'use client';

useEffect(() => {
  if (typeof window !== 'undefined') {
    console.log(window.innerWidth);
  }
}, []);
```

# 他フレームワークでの扱い方

### 1. Nuxt 3

```vue
<template>
  <p v-if="width">Width: {{ width }}px</p>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const width = ref(null);

onMounted(() => {
  width.value = window.innerWidth;
});
</script>
```

> `onMounted` はブラウザ環境でのみ呼ばれるため安全

### 2. SvelteKit

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

> `onMount()` はSvelteKitで「クライアントのみ実行」される構文

### 3. Remix

Remixは基本的にReactと同じですが、`useEffect`の中でクライアント専用処理を行うのが安全です。

```tsx
import { useEffect, useState } from 'react';

export default function WidthViewer() {
  const [width, setWidth] = useState(0);

  useEffect(() => {
    setWidth(window.innerWidth);
  }, []);

  return <p>Width: {width}px</p>;
}
```

> RemixにはServer/Clientの明示的な分離はなく、Clientコードは自己責任で書く必要がある

### 4. Angular

> * Angularは基本的にすべて**クライアント側で動くSPA**
> * **DI（依存性注入）** によってクライアント/サーバーの分岐を柔軟に制御
> * SSRでも、**特定の処理を「クライアント限定」にする必要がある**

Angularで`window`や`document`をそのまま使うと、SSR時にエラーになります。

**❌ NGな例（SSRでクラッシュ）**

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

| フレームワーク     | Server / Client 分離の設計  | クライアント限定コードの扱い方                  |
| ----------- | ---------------------- | -------------------------------- |
| **Next.js** | 明示的（`'use client'`）    | Client Component内で `useEffect` 等 |
| **Nuxt3 (Vue)**| 条件分岐（`process.client`） | `<client-only>` など               |
| **SvelteKit**| 明示的（`onMount`）         | `onMount()` 内に書く                 |
| **Remix**   | 分離なし                   | `useEffect` 等で制御                 |
| **Angular** | 分離なし                   | `isPlatformBrowser()` で明示分岐      |

# おわりに

Next.jsのServer ComponentとClient Componentは、環境の違いを意識しながら設計することで、より安全で効率的なアプリ開発ができるようになります。

特に「**ClientとServerは別の環境**」という視点を持つだけで、

* なぜ`window`が使えないのか
* なぜ`use client`が必要なのか
* なぜHydration Errorが起こるのか

といった疑問の多くが、シンプルに整理できるようになります。

また、今回紹介した他フレームワークとの比較からも、Next.jsがServer Componentを中心に据えたことで、**構文的に安全なアーキテクチャを築いている**ことがわかります。
