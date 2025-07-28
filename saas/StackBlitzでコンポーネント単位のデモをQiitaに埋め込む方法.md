# StackBlitzでコンポーネント単位のデモをQiitaに埋め込む方法

# はじめに

技術記事を書くとき、「動くUIデモをそのままQiita上に埋め込みたい」と思ったことはありませんか？

この記事では、**Next.js (App Router)** と **StackBlitz** を使って、**コンポーネント単位の動作デモをQiita上でインタラクティブに表示する方法**を紹介します。

**🧱 目標**

* ✅ StackBlitzでNext.jsプロジェクトを作成
* ✅ 各コンポーネントをルート単位で分離 (`/scroll-example`, `/button`, etc.)
* ✅ Qiitaに動作デモをiframeで埋め込む

# 1. GitHub に Next.js プロジェクトを作成する

まずは、GitHubリポジトリを事前に用意して、StackBlitzで `https://stackblitz.com/github/ユーザー名/リポジトリ名` にアクセスすることで、StackBlitz上で編集できます。

```bash
npx create-next-app@latest my-demo
cd my-demo
git init
git remote add origin https://github.com/yourname/my-demo.git
git push -u origin main
```

> 💡 今回は以下を使用しています。

https://stackblitz.com/github/oharu121/react-demo

# 2. コンポーネントごとにページを分離

Next.js App Router では、`/app/ディレクトリ名/page.tsx` という構成にすることで、ルートパスごとにページを分けられます。

```bash
app/
├── page.tsx                   ← インデックスページ
├── scroll-example/page.tsx   ← ScrollExampleを表示
├── button/page.tsx           ← Buttonを表示
```

# 3. ルーティングで各コンポーネントに遷移

`app/page.tsx` に、各コンポーネントページへのリンクを追加します。

```tsx:app/page.tsx
import Link from "next/link";

export default function Page() {
  return (
    <main>
      <h1>コンポーネント一覧</h1>
      <ul>
        <li><Link href="/scroll-example">ScrollExample</Link></li>
        <li><Link href="/button">Button</Link></li>
      </ul>
    </main>
  );
}
```

# 4. StackBlitz で GitHub リポジトリを開く

* Githubを連携し、リポジトリを開く

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/b128f503-875f-4466-a388-5bf2d67c2f6b.png)

> 🚨 最初は `https://stackblitz.com/~/github.com/...` という形になりますが、これは後で修正します（次のステップ）。

* StackBlitz上でプロジェクトが開いたら、**URLバーの `/~/` を削除**してエンターを押してください。

🟥 変更前：

```
https://stackblitz.com/~/github.com/oharu121/react-demo?file=app/page.tsx
```

✅ 変更後：

```
https://stackblitz.com/github/oharu121/react-demo?file=app/page.tsx
```

これにより、他人と共有可能な安定したURLになります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/0f006847-e026-46bb-b011-da8e124570e8.png)


# 5. StackBlitzのiframeを埋め込む

StackBlitz では、URLにクエリパラメータ `initialpath=/ルート名` をつけることで、**初期表示されるページを指定**できます。

* 左上の `Share` → `Embed` をクリックし、表示された `iframe` コードをコピーします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/e08a8bdb-906f-4c5c-a62b-d348bbe7ed6d.png)

* iframeをそのままQiitaに埋め込む

```md
<iframe src="https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FScrollExample.tsx&hideExplorer=1&hideNavigation=1&initialpath=/scroll-example&view=preview" width="100%" height="500px" />
```
* `file=components/ScrollExample.tsx` → 解説用にScrollExampleのソースを表示
* `initialpath=/scroll-example` → 実際に `/scroll-example` ページを動かして表示
* `hideExplorer=1` → 左側のファイルツリー非表示
* `hideNavigation=1` → 上部のナビゲーション非表示
* `view=preview` → プレビューのみ表示


✅ Qiitaでも正しく表示されることを確認済み！

<iframe src="https://stackblitz.com/github/oharu121/react-demo?embed=1&file=components%2FScrollExample.tsx&hideExplorer=1&hideNavigation=1&initialpath=/scroll-example&view=preview" width="100%" height="500px" />

# まとめ

* StackBlitzとNext.jsの相性は非常によく、GitHubベースでメンテも容易
* `initialpath`を使えば、任意のページを直接プレビュー可能
* Qiitaでも**iframeを使ったインタラクティブなデモ**が実現できる！

**🎯 最終的にできたもの**

* 各コンポーネントを `/component-name` に分離し、
* StackBlitz経由で個別にデモ表示ができる構成に！
* Qiitaに実際に埋め込んで動作確認済み。
