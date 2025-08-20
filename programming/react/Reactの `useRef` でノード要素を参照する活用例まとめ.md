## React の `useRef` でノード要素を参照する活用例まとめ

## はじめに

React で DOM 要素を直接操作したいとき、`useRef` フックを使うことで対象ノードにアクセスすることができます。本記事では、以下のような代表的なユースケースを実例コードとともに紹介します。

## 1. スクロール操作

> ボタンで特定のセクションに移動

```tsx:ScrollExample.tsx
import React, { useRef } from "react"

export default function ScrollExample() {
  const part1Ref = useRef<HTMLDivElement>(null)
  const part2Ref = useRef<HTMLDivElement>(null)
  const part3Ref = useRef<HTMLDivElement>(null)

  const handleScroll = (ref: React.RefObject<HTMLDivElement>) => {
    ref.current?.scrollIntoView({ behavior: "smooth" })
  }

  return (
    <>
      <div style={{ position: "fixed", background: "#fff", zIndex: 10 }}>
        <button onClick={() => handleScroll(part1Ref)}>第一部</button>
        <button onClick={() => handleScroll(part2Ref)}>第二部</button>
        <button onClick={() => handleScroll(part3Ref)}>第三部</button>
      </div>
      <div ref={whiteRef} style={{ height: "100vh", background: "white" }} />
      <div ref={yellowRef} style={{ height: "100vh", background: "yellow" }} />
      <div ref={blackRef} style={{ height: "100vh", background: "black" }} />
    </>
  )
}
```

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FScrollExample.tsx&hideExplorer=1&hideNavigation=1&initialpath=/scroll-example&view=preview)

🔍 **解説：**

> - `ref` を使うことで、各セクション（第一部/第二部/第三部）の DOM ノードに直接アクセスできます。
> - `scrollIntoView()` メソッドは、その要素が画面に見えるようにスクロールしてくれる便利な API です。**class 名や id に依存しない** ので、DOM 構造が変わっても安心してスクロールできます。

## 2. 初期フォーカスの設定

> ページを開いたら自動で入力欄にカーソルを合わせる``

```tsx:AutofocusInput.tsx
import React, { useEffect, useRef } from "react"

export default function AutofocusInput() {
  const inputRef = useRef<HTMLInputElement>(null)

  useEffect(() => {
    inputRef.current?.focus()
  }, [])

  return (
    <div>
      <label>Email:</label>
      <input ref={inputRef} type="email" placeholder="your@email.com" />
    </div>
  )
}
```

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FAutofocusInput.tsx&hideExplorer=1&hideNavigation=1&initialpath=/autofocus-input&view=preview)

🔍 **解説：**

> - フォームで「ページを開いたら自動的に入力欄にフォーカスさせたい」ことはよくあります。
> - `useRef` で input 要素を取得し、`useEffect`で初回マウント時に `.focus()` を呼べば、実現可能です。React 的にフォーカスを操作したいときはこのパターンが定番です。

## 3. Video プレイヤーの再生・停止制御

```tsx:VideoPlayer.tsx
import React, { useRef, useState } from "react"

export default function VideoPlayer() {
  const videoRef = useRef<HTMLVideoElement>(null)
  const [isPlaying, setIsPlaying] = useState(false)

  const togglePlay = () => {
    if (!videoRef.current) return
    isPlaying ? videoRef.current.pause() : videoRef.current.play()
    setIsPlaying((prev) => !prev)
  }

  return (
    <div>
      <video ref={videoRef} width={320}>
        <source src="https://www.w3schools.com/html/mov_bbb.mp4" type="video/mp4" />
      </video>
      <button onClick={togglePlay}>{isPlaying ? "Pause" : "Play"}</button>
    </div>
  )
}
```

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FVideoPlayer.tsx&hideExplorer=1&hideNavigation=1&initialpath=/video-player&view=preview)

🔍 **解説：**

> - `ref` を使って `<video>` 要素の `.play()` や `.pause()` を直接呼び出すことで、状態に応じた制御が可能になります。HTML5 のビデオ要素はコントロールが複雑になりやすいので、`useRef` 経由で直接操作できると便利です。

## 4. モーダル外クリックで閉じる

```tsx:ClickOutsideModal.tsx
import React, { useEffect, useRef } from "react"

interface Props {
  onClose: () => void
}

export default function ClickOutsideModal({ onClose }: Props) {
  const modalRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const handleClick = (e: MouseEvent) => {
      if (modalRef.current && !modalRef.current.contains(e.target as Node)) {
        onClose()
      }
    }

    document.addEventListener("mousedown", handleClick)
    return () => document.removeEventListener("mousedown", handleClick)
  }, [onClose])

  return (
    <div ref={modalRef} style={{ background: "#fff", padding: "2rem" }}>
      <h2>モーダル</h2>
      <p>外側をクリックすると閉じます</p>
    </div>
  )
}
```

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FClickOutsideModal.tsx&hideExplorer=1&hideNavigation=1&initialpath=/click-outside-modal&view=preview)

🔍 **解説：**

> - `ref` を使うことで「自分自身がクリックされたかどうか」を判定できます。
> - React では `document.addEventListener` を使って外側のクリックを検知し、`ref.current.contains()` でモーダル外かを判断します。モーダルやドロップダウンの定番パターンです。

## 5\. フィールドノート（メモチャット）

> このコンポーネントは、簡易的なチャット風のメモアプリです。

```tsx:FieldNotes.tsx
import * as React from "react"

export default function FieldNotes() {
  const [notes, setNotes] = React.useState([
    "ReactコンポーネントはUIの見た目とロジックをまとめて管理します。",
    "関数を組み合わせるように、コンポーネントも組み合わせてUIを作成できます。",
    "JSXはJavaScriptの力とHTMLの読みやすさを融合しています。",
    "フックを使うことで、ロジックの再利用が簡単になります。",
  ])

  const newNoteRef = React.useRef<HTMLLIElement>(null)

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    const form = e.target as HTMLFormElement
    const formData = new FormData(form)
    const newNote = formData.get("note")

    if (typeof newNote === "string" && newNote.trim()) {
      setNotes([...notes, newNote])
      form.reset()
    }
  }

  React.useEffect(() => {
    if (newNoteRef.current) {
      newNoteRef.current.scrollIntoView({ behavior: "smooth" })
    }
  })

  return (
    <div>
      <article>
        <h1>Field Notes</h1>
        <div>
          <ul>
            {notes.map((msg, index) => (
              <li
                ref={index === notes.length - 1 ? newNoteRef : null}
                key={index}
              >
                {msg}
              </li>
            ))}
          </ul>
          <form onSubmit={handleSubmit}>
            <input
              required
              type="text"
              name="note"
              placeholder="メモを入力してください..."
            />
            <button type="submit">
              送信
            </button>
          </form>
        </div>
      </article>
    </div>
  )
}
```

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FFieldNotes.tsx&hideExplorer=1&hideNavigation=1&initialpath=/field-notes&view=preview)

🔍 **解説：**

> - `useState` でメモのリストを管理し、`useRef` で最新メモの DOM 要素を参照します。
> - メモを追加すると、`scrollIntoView({ behavior: \"smooth\" })` で自動的に最新メモへスクロールします。
> - フォーム送信時に新しいメモを追加し、空欄や空白のみの入力は無視します。

Certainly! Here’s an article section for your FollowTheLeader component, stripped of styling and following your requested format:

## 6\. クリックした位置にマーブルが移動する

> クリックした位置にマーブル（円）がスムーズに移動します。

```tsx:FollowTheLeader.tsx
import React, { useEffect, useRef, useState } from "react"

export default function FollowTheLeader() {
  const getCenter = (): [number, number] =>
    [window.innerWidth / 2 - 32, window.innerHeight / 2 - 32]
  const [position, setPosition] = useState<[number, number]>([0, 0])
  const boxRef = useRef<HTMLDivElement>(null)
  const [initialized, setInitialized] = useState(false)

  useEffect(() => {
    setPosition(getCenter())
    setInitialized(true)
    const handleResize = () => setPosition(getCenter())
    window.addEventListener('resize', handleResize)
    return () => window.removeEventListener('resize', handleResize)
  }, [])

  const handleClick = (event: React.MouseEvent<HTMLDivElement>) => {
    if (!boxRef.current) return
    const { width, height } = boxRef.current.getBoundingClientRect()
    setPosition([
      event.clientX - width / 2,
      event.clientY - height / 2,
    ])
  }

  return (
    <div onClick={handleClick}>
      <div
        ref={boxRef}
        style={{
          position: "absolute",
          left: 0,
          top: 0,
          width: 64,
          height: 64,
          borderRadius: "50%",
          transform: `translate(${position[0]}px, ${position[1]}px)`,
          opacity: initialized ? 1 : 0,
        }}
      />
    </div>
  )
}
```

@[stackblitz](https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FFollowTheLeader.tsx&hideExplorer=1&hideNavigation=1&initialpath=/follow-the-leader&view=preview)

🔍 **解説：**

> - `useRef` でマーブルの DOM 要素を参照し、クリック時の座標からマーブルの中心がクリック位置に来るように座標を計算しています。
> - ウィンドウリサイズ時も中央に再配置されるように `useEffect` でイベントリスナーを追加しています。

**まとめ：**

| 用途           | 操作対象              | 解説                     |
| -------------- | --------------------- | ------------------------ |
| スクロール移動 | `.scrollIntoView()`   | 特定要素への移動が簡単に |
| 自動フォーカス | `.focus()`            | UX 向上に必須            |
| メディア制御   | `.play()`, `.pause()` | 状態に応じた再生切替     |
| 外クリック検知 | `.contains()`         | モーダルやポップアップに |
| 表示検知       | IntersectionObserver  | アニメや LazyLoad に便利 |
| スクロール復元 | `.scrollTop`          | リロード時の UX 向上     |

## 終わりに

`useRef` を使えば、React でも DOM 操作が必要な場面に柔軟に対応できます。特に次のような場面で効果を発揮します：

- 初期化時の操作（フォーカス、スクロール）
- メディア制御
- モーダルやドロップダウンなどの UI 操作
- Intersection Observer を使った表示判定
