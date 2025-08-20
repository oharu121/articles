## CSS アニメーションが終わるまでトグルを無効化する React パターン

## はじめに

React でトグル UI を実装する際、こんな経験はありませんか？

> 「ボタンを連打したら、CSS アニメーションが中途半端に切り替わって表示が崩れてしまった…」

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FBrokenAnimationBox.tsx&hideExplorer=1&hideNavigation=1&initialpath=/broken-animation-box&view=preview)

これは、React の状態変更の速さと、CSS アニメーションの再生時間にズレがあるために起こる典型的な問題です。

この記事では、**`Element.getAnimations()`** というモダンな Web API を使い、**CSS アニメーションの完了を正確に検知してトグルの連打を防止する**、堅牢な実装パターンを紹介します。

**❌ なぜ`setTimeout`では不十分なのか？**

従来、アニメーション時間に合わせて`setTimeout`で入力をロックする手法が使われていました。

```ts
const [isDisabled, setIsDisabled] = useState(false);

const handleClick = () => {
  if (isDisabled) return;

  setIsDisabled(true); // ボタンを無効化
  setActive((prev) => !prev);

  // CSSのアニメーション時間（例: 500ms）に合わせてロックを解除
  setTimeout(() => {
    setIsDisabled(false);
  }, 500);
};
```

この方法は一見シンプルですが、大きな欠点があります。

> - **保守性の低下**: CSS でアニメーション時間を変更するたびに、JavaScript のコードも修正する必要がある。
> - **不正確さ**: 複雑なアニメーションやブラウザの負荷状況によっては、実際の終了時間とズレが生じる可能性がある。

これらの問題を解決するのが、**`Element.getAnimations()`** API です。

## 解決策：`getAnimations()` でアニメーションを監視する

`Element.getAnimations()` は、指定した DOM 要素とその子孫要素で **現在実行中のすべてのアニメーション** の情報を取得できる Web API です。

各アニメーションオブジェクトは `.finished` というプロパティを持っており、これは**アニメーションの完了時に解決される `Promise`** です。

これを利用することで、「**すべてのアニメーションが終わるまで待つ**」という処理を、`setTimeout`に頼らず、正確かつ宣言的に実装できます。

## 1\. `useRef`でアニメーションの状態を管理する

まず、アニメーション中かどうかを判定するフラグと、アニメーション対象の DOM 要素への参照を用意します。

```tsx:MultiAnimation.tsx
const boxRef = useRef<HTMLDivElement>(null);
const isAnimatingRef = useRef(false);
```

ここで重要なのは、アニメーションの状態管理に`useState`ではなく`useRef`を使う点です。

> - **`useState`**: 値が変更されるたびに**再レンダリングがトリガー**されます。
> - **`useRef`**: 値が変更されても再レンダリングは発生しません。

「アニメーション中かどうか」という状態は、UI の見た目に直接影響するものではなく、内部的なロジックのためのフラグです。`useRef`を使うことで、**不要な再レンダリングを防ぎ、パフォーマンスを最適化**できます。

## 2\. `useEffect`でアニメーションの完了を待つ

トグルの状態 `active` が変更されたことをトリガーに、`useEffect` を使ってアニメーションの監視を開始します。

```tsx:MultiAnimation.tsx
useEffect(() => {
  const node = boxRef.current;
  if (!node) return;

  // アニメーション監視用の非同期関数
  const checkAnimationEnd = async () => {
    // 1. まず、アニメーション中フラグを立ててトグルをロック
    isAnimatingRef.current = true;

    // 2. 実行中のすべてのアニメーションを取得し、それらの完了Promiseの配列を作成
    const animations = node.getAnimations({ subtree: true });
    const promises = animations.map(anim => anim.finished);

    // 3. すべてのアニメーションが完了するのを待つ
    await Promise.all(promises);

    // 4. すべて完了したら、フラグを下げてロックを解除
    isAnimatingRef.current = false;
  };

  checkAnimationEnd();
}, [active]); // active（トグルのON/OFF）が変わるたびに実行
```

この`useEffect`が、React の **状態（`active`）** と、ブラウザの **外部システム（CSS アニメーション）** とを同期させる役割を担います。

## 3\. クリックハンドラでロックを判定

最後に、クリックハンドラで`isAnimatingRef`フラグをチェックするだけのシンプルな実装です。

```tsx:MultiAnimation.tsx
const handleToggle = () => {
  // isAnimatingRefがtrue（アニメーション中）なら、何もしない
  if (isAnimatingRef.current) return;

  setActive(prev => !prev);
};
```

**完成形**

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FSafeAnimationBox.tsx&hideExplorer=1&hideNavigation=1&initialpath=/safe-animation-box&view=preview)

## 🚀 【応用】カスタムフックでロジックを再利用可能にする

このアニメーションロックのロジックは、様々なコンポーネントで再利用できます。そこで、`useAnimationLock`というカスタムフックに切り出してみましょう。

```ts:useanimationlock.ts
import { useRef, useEffect, RefObject } from 'react';

export function useAnimationLock<T extends HTMLElement>(
  targetRef: RefObject<T>,
  dependencies: any[]
) {
  const isAnimatingRef = useRef(false);

  useEffect(() => {
    const node = targetRef.current;
    if (!node) return;

    const checkAnimationEnd = async () => {
      isAnimatingRef.current = true;
      const animations = node.getAnimations({ subtree: true });
      await Promise.all(animations.map(anim => anim.finished));
      isAnimatingRef.current = false;
    };

    checkAnimationEnd();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, dependencies);

  return isAnimatingRef;
}
```

**コンポーネント側のコード**

カスタムフックを使うことで、コンポーネント本体は非常にシンプルで宣言的になります。

```tsx:Togglebox.tsx
import { useState, useRef } from 'react';
import { useAnimationLock } from './useAnimationLock';

function ToggleBox() {
  const [active, setActive] = useState(false);
  const boxRef = useRef<HTMLDivElement>(null);

  // カスタムフックを呼び出すだけ！
  const isAnimatingRef = useAnimationLock(boxRef, [active]);

  const handleToggle = () => {
    if (isAnimatingRef.current) return;
    setActive(prev => !prev);
  };

  return (
    <>
      <button onClick={handleToggle} disabled={isAnimatingRef.current}>
        Toggle
      </button>
      <div
        ref={boxRef}
        className={`box ${active ? 'animate-on' : 'animate-off'}`}
      />
    </>
  );
}
```

## まとめ

React と CSS アニメーションを正しく同期させるためのベストプラクティスは以下の通りです。

> - **`setTimeout`は使わない**: `Element.getAnimations()` API を使い、アニメーションの完了を確実に検知する。
> - **`useRef`を賢く使う**: UI の再レンダリングが不要な内部的な状態（フラグなど）は、`useRef`で管理してパフォーマンスを最適化する。
> - **副作用は`useEffect`に集約する**: React の状態変更をトリガーに外部システム（DOM API など）と連携する処理は、`useEffect`の責務。
> - **カスタムフックでロジックを共通化する**: 再利用可能なロジックはカスタムフックに切り出し、コンポーネントをクリーンに保つ。

これらのパターンを活用することで、ユーザー体験を損なわない、堅牢な UI を構築できます。
