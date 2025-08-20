## はじめに

`npm run` は、プロジェクトの共通タスク（ビルド／テスト／デプロイ前処理など）をチームで統一する最強の入口です。

中でも **`pre:` と `post:` のフック**を使うと、「あるスクリプトの前後で必ず実行される処理」をシンプルに組み立てられます。この記事では：

>* pre/post の基本と実行順序
>* 代表的なライフサイクルフック（install, prepare, test, build…）
>* 実務で役立つレシピ集（lint → build → test の自動連鎖、バージョンタグ自動付与、環境変数の扱い など）
>* クロスプラットフォーム対応や落とし穴

をわかりやすく解説します。

## 基本：`scripts` と `npm run`

`package.json` の `scripts` は、`npm run <name>` で呼び出せるコマンドの定義です。

```jsonc
{
  "scripts": {
    "start": "node dist/server.js",
    "build": "tsc -p tsconfig.json",       // TypeScript のビルド
    "test": "node --test",                 // Node.js組み込みテスト or jest/mocha等
    "lint": "eslint .",
    "dev": "tsx watch src/index.ts"
  }
}
```

**📌 ポイント**

>* `npm run build` の省略形で `npm build` が使える **予約名** もあります（`start`, `stop`, `test` など）。
>* スクリプト内では `node_modules/.bin` が **自動で PATH に入る**ため、ローカル依存のCLIをフルパスなしで実行できます。

## `pre` / `post` の動き

任意の `scripts` に対し、名前の前に `pre` / 後に `post` を付けるだけで **前後フック**が作れます。

| 実行コマンド          | 実行順序                               |
| --------------- | ---------------------------------- |
| `npm run build` | `prebuild` → `build` → `postbuild` |
| `npm run test`  | `pretest` → `test` → `posttest`    |
| `npm run start` | `prestart` → `start` → `poststart` |

```jsonc
{
  "scripts": {
    "prebuild": "rimraf dist",              // ① 前処理：出力物の掃除
    "build": "tsc -p tsconfig.json",        // ② 本体：ビルド
    "postbuild": "node scripts/size.js"     // ③ 後処理：サイズ計測など
  }
}
```

**📌 ポイント**

>* `pre<name>` が失敗（非0終了）すると `<name>` と `post<name>` は実行されません。
>* `<name>` が成功して `post<name>` で失敗した場合、コマンド全体は失敗扱いです（CIで落とせる）。

## npm ライフサイクルスクリプト

`npm` には、特別なタイミングで自動実行される **ライフサイクル** があります。よく使うものだけ覚えると実務は十分回ります。

| フック                                      | いつ走る？                             | 主用途                                |
| ---------------------------------------- | --------------------------------- | ---------------------------------- |
| `preinstall` / `install` / `postinstall` | 依存関係インストール時                       | ネイティブビルド、環境チェック、生成物の配置             |
| `prepare`                                | `npm install` や `npm pack` の直前    | Git からクローンした直後にビルド・生成が必要なパッケージで便利  |
| `prepublishOnly`                         | `npm publish` の直前（**publish時のみ**） | 公開直前の最終チェックやビルド                    |
| `preversion` / `version` / `postversion` | `npm version` 実行時                 | バージョン更新に紐づく自動処理（changelog生成、タグ付け等） |
| `pretest` / `test` / `posttest`          | `npm test`                        | テスト前後の初期化・片付け                      |
| `prebuild` / `build` / `postbuild`       | 任意                                | ビルド前後での掃除や成果物の検査                   |

**🪄 豆知識**

> `prepare` は Git から直接インストールされた時にも走るため、**配布物に `dist/` を含めないライブラリ**で重宝します。逆にアプリでは不要なことが多く、`postinstall` の方が分かりやすい場合もあります。

## 実務レシピ集

### 1\. Lint → Build → Test を自動連鎖

```jsonc
{
  "scripts": {
    "prebuild": "eslint . && rimraf dist",
    "build": "tsc -p tsconfig.json",
    "postbuild": "node scripts/size.js",

    "pretest": "npm run build",
    "test": "node --test", // or "jest"
    "posttest": "node scripts/report-coverage.js"
  }
}
```

**📌 ポイント**

>* `npm test` だけで、**lint → clean → build → test → coverage** まで一直線。

### 2\. バージョンアップ時に自動で CHANGELOG と Git タグ

```jsonc
{
  "scripts": {
    "preversion": "npm run test",
    "version": "node scripts/changelog.js && git add CHANGELOG.md",
    "postversion": "git push && git push --tags"
  }
}
```

**📌 ポイント**

>* `npm version patch` → テスト成功なら `CHANGELOG.md` 更新 → コミット＆タグを自動プッシュ。

### 3\. インストール時に生成物を用意

```jsonc
{
  "scripts": {
    "postinstall": "node scripts/postinstall.js",
    "prepare": "npm run build" // ライブラリの場合のみ推奨
  }
}
```

**📌 ポイント**

>* バイナリのダウンロード、テンプレ作成、型生成などを `postinstall` で。

### 4\. デプロイ前に安全弁

```jsonc
{
  "scripts": {
    "predeploy": "npm run build && node scripts/verify-env.js",
    "deploy": "node scripts/deploy.js",
    "postdeploy": "echo '✅ Deploy Done'"
  }
}
```

**📌 ポイント**

>* `predeploy` で環境変数や認証を検証。失敗したらデプロイ自体が止まる。

## 便利テク

### 1\. 変数を使い回す

`npm run` 実行時は、環境変数 `npm_lifecycle_event`（今走っているスクリプト名）や `npm_package_*`（package.jsonの値）が使えます。

```sh
# scripts 内から
echo "running=$npm_lifecycle_event name=$npm_package_name version=$npm_package_version"
```

### 2\. クロスプラットフォーム対応

Windows / macOS / Linux で同じ動きをさせたい場合、以下のユーティリティが便利です：

>* `cross-env`：環境変数のセット（例：`cross-env NODE_ENV=production node app.js`）
>* `rimraf`：`rm -rf` 互換の削除
>* `shx`：`mkdir`, `cp` などのシェルコマンド互換
>* `npm-run-all`：直列/並列実行（`run-s`, `run-p`）

```jsonc
{
  "scripts": {
    "clean": "rimraf dist",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "lint": "eslint .",
    "build": "cross-env NODE_ENV=production tsc -p tsconfig.json",
    "check": "npm-run-all -s clean lint typecheck build" // 直列実行
  }
}
```

### 3\. ワークスペースでの共通コマンド

ルートで `npm run -ws <script>` とすると、各パッケージの同名スクリプトを一括実行できます。

```sh
npm run -ws build     # すべてのワークスペースで build を実行
npm run -ws test --if-present
```

### 4\. Git フック連携（Husky）

コミット前に `npm test` を強制するなど、pre-commit フックに `npm run` を組み合わせると品質が安定します。

**‼️ 落とし穴**

>* **無限ループに注意**：`pre<name>` の中でうっかり `npm run <name>` を呼ぶとループします。
>* **過剰なネスト**：pre/post が深くなりすぎると依存関係が見えづらくなります。重要な流れは **明示的な複合スクリプト**（例：`npm-run-all`）で管理を。
>* **`prepare` の乱用**：アプリで `prepare` を使うと、`npm install` のたびに重いビルドが走ってストレスに。**配布ライブラリ中心で使う**意識を。
>* **プラットフォーム差**：`rm`, `export` など UNIX 前提の記述は Windows で落ちます。`rimraf` や `cross-env` を使いましょう。

**👍 ベストプラクティス**

>1\. \*\*「主タスク + pre/post」\*\*で三層に整理（clean → main → verify）。
>2\. CI では **`npm ci`**（再現性◎）を使い、`postinstall` の重い処理は避ける／条件分岐する。
>3\. 重要なフローは **`check` や `verify` の総合タスク**にまとめ、開発者は原則その一発でOK。
>4\. スクリプト名は **動詞の命令形**で統一（`build`, `test`, `deploy`, `release` など）。
>5\. **環境差を吸収**（`cross-env`, `rimraf`, `npm-run-all`）して「動くプロジェクト」に。

**🎠 サンプル `package.json`**

```jsonc
{
  "name": "awesome-app",
  "version": "1.2.3",
  "private": true,
  "scripts": {
    // 開発
    "dev": "tsx watch src/index.ts",

    // 品質チェック
    "lint": "eslint .",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "format": "prettier -w .",
    "check": "npm-run-all -s clean lint typecheck build test",

    // ビルド
    "prebuild": "rimraf dist",
    "build": "cross-env NODE_ENV=production tsc -p tsconfig.json",
    "postbuild": "node scripts/size.js",

    // テスト
    "pretest": "npm run build",
    "test": "node --test --test-reporter spec",
    "posttest": "node scripts/report-coverage.js",

    // デプロイ
    "predeploy": "npm run check && node scripts/verify-env.js",
    "deploy": "node scripts/deploy.js",
    "postdeploy": "echo \"✅ Deploy finished for $npm_package_name@$npm_package_version\"",

    // リリース（バージョン更新）
    "preversion": "npm test",
    "version": "node scripts/changelog.js && git add CHANGELOG.md",
    "postversion": "git push && git push --tags"
  },
  "devDependencies": {
    "cross-env": "^7.0.3",
    "eslint": "^9.0.0",
    "npm-run-all": "^4.1.5",
    "prettier": "^3.3.0",
    "rimraf": "^6.0.0",
    "tsx": "^4.15.0",
    "typescript": "^5.6.0"
  }
}
```

## まとめ

* [ ] pre/post でタスクの前後関係をシンプルに表現できている
* [ ] `prepare` / `postinstall` は用途を理解して使い分けた
* [ ] CI は `npm ci`、ローカルは `npm install` で運用
* [ ] クロスプラットフォーム対応（`cross-env` / `rimraf` / `npm-run-all`）
* [ ] 総合チェックコマンド（`check` or `verify`）を用意してチームの入り口にする
