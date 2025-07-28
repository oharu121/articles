# GitHubでの基本的な開発フローを理解する

# はじめに

GitHubは現代のソフトウェア開発において不可欠なツールです。本記事では、git clone, git branch, git push, git pull などの基本的なコマンドを使った実践的な開発フローを解説します。これらのコマンドの理解と適切な使用は、効率的な開発とスムーズなチーム協働の鍵となります。


**典型的なワークフローの例**

1/. リポジトリのクローン（git clone）
2/. 新規フィーチャーブランチの作成（git branch）
3/. 開発と変更のコミット（git commit）
4/. リモートへの変更の共有（git push）
5/. プルリクエストの作成とレビュー
6/. メインブランチへのマージ
7/. ローカルの更新（git pull）
8/. 不要ブランチの整理

既存プロジェクトをgithubにアップロードする場合、下記をご参照ください。

https://qiita.com/oharu121/items/5439c843d0ca1bb83448

# リポジトリをセットアップする

### 1. GitHubでSSH鍵の設定

SSH鍵の生成方法は、下記の記事をご参照ください。

https://qiita.com/oharu121/items/1ca0053c8025d4ca72dc

公開鍵（C:\Users\<ユーザ名>\.ssh\id_rsa.pub）をGitHubの設定画面に追加

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/8ebe48a2-f946-c5db-76ba-fe382c3cbc5e.png)

### 2. git clone を実行する

git clone コマンドは、リモートリポジトリをローカルマシンにコピーするために使用されます。以下の手順に従ってください：

1. GitHubからリポジトリのURLをコピーします
1. ローカルで作業するディレクトリをターミナルで開きます（例: E:\Script_testing）
1. 下記のコマンドを実行してリポジトリをクローンします：
```bash
git clone https://github.com/username/example-repository.git
```

**📌 ポイント**
>* git clone はプロジェクトフォルダを自動的に作成するため、事前にフォルダを作る必要はありません。
>* URLの末尾に .git を付けることを忘れずに。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/725efbe9-78dc-f3b8-7746-4d6c0a71b7dd.png)


# 機能開発の流れ

今mainのファイル構成は下記としましょう：

```shell
example-repository/
├── package-lock.json
├── package.json
└── tsconfig.json
```

### 1. 新しいブランチを作成する

デフォルトのブランチは main ですが、直接 main を修正するのは推奨されません。新しいフィーチャーブランチを作成して、そこで作業を行います。
```bash
git checkout -b "feature/login-page"
```

このコマンドで、新しいブランチ「feature/login-page」が作成され、そのブランチに切り替わります。ここで Login.tsx を実装すると、以下のようにプロジェクト構成が変更されます：

```diff shell
example-repository/
+ ├── Login.tsx
├── package-lock.json
├── package.json
└── tsconfig.json
```

別の機能（例えば、news 機能）を開発する場合は、main ブランチに戻ります。

```bash
git checkout main
```

そうすると、構成がもとの状態に戻ります。

```shell
example-repository/
├── package-lock.json
├── package.json
└── tsconfig.json
```

新しいブランチを作成して作業を進めます。

```bash
git checkout -b "feature/news-page"
```

このコマンドで、新しいブランチ「feature/news-page」が作成され、そのブランチに切り替わります。ここで News.tsx を実装すると、以下のようにプロジェクト構成が変更されます：

```diff shell
example-repository/
+ ├── News.tsx
├── package-lock.json
├── package.json
└── tsconfig.json
```
この方法により、各機能は独立したブランチで管理され、互いに影響を与えることなく並行して開発できます。

### 2. 修正をコミットする

変更が完了したら、次のコマンドで修正をステージングします。

```bash
git add .
```

次のコマンドで修正をコミットします。

```bash
git commit -m "Newsページの実装"
```

コミット後、リモートリポジトリに変更をプッシュします。

```bash
git push origin ブランチ名
```

下記コマンドを実行するとブランチ名を指定しなくても、現在チェックアウトしているブランチが自動的にプッシュされます。

```bash
git config --global push.default current
```

# Githubでマージする

### 1. プルリクエストを確認する

リモートにプッシュすると、GitHubで自動的にプルリクエストが生成されます。チームメンバーが変更を確認し、コンフリクトがないかを確認します。

### 2. ブランチをマージする

コードレビューが完了し、問題がなければ、変更を main ブランチにマージします。

# ロカールを整理する

### 1. mainブランチに切り替える

次のコマンドでmain ブランチに戻ります。

```bash
git checkout main
```

mainブランチはこの時点まだLogin機能がまだ反映されていないため、次のコマンドでGitHubから最新の変更を取得します。

```bash
git pull
```

### 2. 不要なブランチを削除する

使用済みのブランチを削除して、ローカルリポジトリを整理します。

```bash
git branch -d feature/login-page
```

### 3. 次の機能を開発する

引き続き、Loginページの実装に戻ります。

```bash
git checkout feature/login-page
```

git branchですべてのブランチを確認できます。

# まとめ

これらの基本的なGitコマンドを理解することで、より効率的にチームでの開発を進めることができます。さらに高度なGitの機能についても学習し、活用してみてください。