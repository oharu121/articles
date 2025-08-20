## Husky で始める Git フック自動化：コード品質を保つ Node.js プロジェクトの実装ガイド

## はじめに

「コミット前にテストを実行し忘れた...」「ESLint エラーがあるコードを間違えてプッシュしてしまった...」こんな経験はありませんか？

チーム開発では、**コード品質を一定に保つ仕組み**が重要です。毎回手動でテストやリントを実行するのは現実的ではありませんし、人的ミスも避けられません。

そこで役立つのが **Husky** です。Git フックを簡単に設定して、コミットやプッシュのタイミングで自動的にチェックを実行できます。

**🎯 この記事で実現すること**

>- コミット前に ESLint と Prettier を自動実行
>- プッシュ前にテストを自動実行
>- 問題があれば操作を自動でブロック
>- チーム全体でコード品質を統一

## Husky とは？

**Husky** は、Git フックを Node.js プロジェクトで簡単に管理できるツールです。

**💡 Git フックとは？**

Git が特定のタイミング（コミット前、プッシュ前など）で自動実行するスクリプトのことです。

```bash
# 従来の Git フック（面倒な設定）
.git/hooks/pre-commit  # 手動でシェルスクリプトを作成
.git/hooks/pre-push    # バージョン管理されない
```

**✅ Husky を使うメリット**

| 従来の方法                     | Husky                          |
| ------------------------------ | ------------------------------ |
| `.git/hooks/` に手動でファイル | `package.json` で簡単設定     |
| バージョン管理されない         | チーム全体で共有可能           |
| シェルスクリプトの知識が必要   | npm scripts をそのまま実行可能 |

## Husky のセットアップ

### 1. Husky のインストール

```bash
npm install --save-dev husky
```

### 2. Husky の初期化

```bash
npx husky-init
npm install
```

これで `.husky/` フォルダが作成され、サンプルの `pre-commit` フックが設定されます。

**📁 作成されるファイル構造**

```
project/
├── .husky/
│   ├── _/           # Husky の内部ファイル
│   └── pre-commit   # コミット前に実行されるスクリプト
├── package.json
└── ...
```

### 3. package.json の確認

`husky-init` により、以下が自動追加されます：

```json:package.json
{
  "scripts": {
    "prepare": "husky install"
  },
  "devDependencies": {
    "husky": "^8.0.0"
  }
}
```

**🔍 解説**

>- `prepare` スクリプト：`npm install` 時に自動で `husky install` を実行
>- チームメンバーが `npm install` するだけで Husky が有効になる

## 実用的な Git フック設定例

### 1\. コミット前チェック（pre-commit）

```bash:.husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."

# ESLint でコードチェック
npm run lint

# Prettier でフォーマットチェック
npm run format:check

# TypeScript の型チェック
npm run type-check

echo "✅ Pre-commit checks passed!"
```

### 2\. プッシュ前チェック（pre-push）

```bash:.husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🚀 Running pre-push checks..."

# テスト実行
npm run test

# ビルドが通るかチェック
npm run build

echo "✅ Pre-push checks passed!"
```

### 3\. 対応する package.json scripts

```json:package.json
{
  "scripts": {
    "lint": "eslint . --ext .js,.jsx,.ts,.tsx",
    "lint:fix": "eslint . --ext .js,.jsx,.ts,.tsx --fix",
    "format:check": "prettier --check .",
    "format:fix": "prettier --write .",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "build": "next build"
  }
}
```

## より高度な設定：lint-staged との組み合わせ

**問題点**：コミット時に全ファイルをチェックすると時間がかかる

**解決策**：`lint-staged` でステージされたファイルのみをチェック

### 1\. lint-staged のインストール

```bash
npm install --save-dev lint-staged
```

### 2\. package.json の設定

```json:package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yml,yaml}": [
      "prettier --write"
    ]
  },
  "scripts": {
    "prepare": "husky install"
  }
}
```

### 3\. pre-commit フックの更新

```bash:.husky/pre-commit
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🔍 Running lint-staged..."
npx lint-staged

echo "🧪 Running tests related to staged files..."
npm run test -- --findRelatedTests --passWithNoTests
```

**🎁 メリット**

>- **高速化**：変更されたファイルのみをチェック
>- **自動修正**：ESLint や Prettier が自動で問題を修正
>- **関連テスト**：変更されたファイルに関連するテストのみ実行

## 実際の開発フローでの活用例

### ❌ 問題のあるコミットをブロック

```bash
$ git commit -m "add new feature"

🔍 Running pre-commit checks...
✖ ESLint found 3 errors:
  src/components/Button.tsx:15:10 - Missing semicolon
  src/utils/helper.ts:8:1 - Unused variable 'temp'

error Command failed with exit code 1.
husky - pre-commit hook exited with code 1 (error)
```

→ **コミットが自動でキャンセルされ、問題の修正が促される**

### ✅ 正常なコミットの流れ

```bash
$ git commit -m "add new feature"

🔍 Running lint-staged...
✔ eslint --fix
✔ prettier --write
✔ git add

🧪 Running tests related to staged files...
 PASS  src/components/Button.test.tsx
 PASS  src/utils/helper.test.ts

✅ Pre-commit checks passed!

[main abc1234] add new feature
 2 files changed, 15 insertions(+), 3 deletions(-)
```

## チーム導入時のベストプラクティス

### 1. 段階的な導入

```json:package.json
{
  "scripts": {
    "prepare": "husky install || true"
  }
}
```

**🔍 解説**

>- `|| true` により、Husky の設定に失敗してもインストールは継続
>- 既存プロジェクトに導入する際の安全策

### 2. 設定のカスタマイズ例

```bash:.husky/commit-msg
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# コミットメッセージの形式チェック
npx commitlint --edit $1
```

```json:package.json
{
  "devDependencies": {
    "@commitlint/cli": "^17.0.0",
    "@commitlint/config-conventional": "^17.0.0"
  }
}
```

### 3. 緊急時のスキップ方法

```bash
# Gitフックをスキップしてコミット（緊急時のみ）
git commit -m "hotfix" --no-verify

# または環境変数で無効化
HUSKY=0 git commit -m "emergency fix"
```

**‼️ よくある問題と解決法**

| 問題                             | 解決方法                                        |
| -------------------------------- | ----------------------------------------------- |
| `husky command not found`        | `npm run prepare` を実行                       |
| フックが実行されない             | `.husky/` ファイルの実行権限を確認              |
| Windows で改行コードエラー       | `.gitattributes` で `* text=auto` を設定       |
| CI/CD でフックが重複実行される   | CI 環境では `HUSKY=0` で無効化                  |

**‼️ Windows 環境での注意点**

```bash
# .husky/_/husky.sh のパス区切り文字を確認
# Windows では Git Bash または WSL の使用を推奨
```

## おわりに

Husky を導入することで、**人的ミスによるコード品質の低下を確実に防げます**。

特に以下のような場面で効果を発揮します：

>- **チーム開発**：全員が同じ品質基準でコードを書ける
>- **CI/CD パイプライン**：デプロイ前の最後の砦として機能
>- **コードレビュー**：基本的な問題を事前に解決し、レビューの質が向上

まずは `pre-commit` でのリントチェックから始めて、徐々にテストやビルドチェックを追加していきましょう。

あなたのプロジェクトもワンランク上のコード品質を実現できるはずです！