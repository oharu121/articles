## WSL 2 と Docker を活用した Ruby 開発環境の構築

## はじめに

Windows 上でネイティブな Linux 環境を提供する **WSL 2** と、軽量な仮想環境を構築できる **Docker** を組み合わせることで、Windows でも快適な Ruby 開発環境を手に入れることができます。

本記事では、以下の流れで WSL 2 + Docker による Ruby 開発環境の構築手順を解説します。

- ✅ WSL 上の Ubuntu を起動
- ✅ Docker + Docker Compose の導入
- ✅ Ruby 実行環境のセットアップ
- ✅ コンテナ内で Ruby を実行

まずは WSL のセットアップが済んでいない方は、以下の記事を参考にしてください 👇

https://zenn.dev/oharu121/articles/d8d5037a1edfd8

## 1. Ubuntu を起動しよう

Windows の検索バーで「**ターミナル**」を入力し、ターミナルを起動

![ターミナルを起動](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/4a6d5787-d1c9-4e57-b370-b5a3aa1856aa.png)

ドロップダウンから「**Ubuntu**」を選択

![Ubuntuを選択](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/8f5159fb-6f86-884f-f6a5-e853507155c6.png)

Ubuntu が立ち上がれば準備完了です！

![Ubuntu起動完了](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/1657ba48-04cf-30f5-2bcb-930e4e9c3eaf.png)

## 2. Ruby 環境を構築しよう（Docker 編）

#### 1. プロジェクトフォルダの作成

まずは作業用ディレクトリを作成します。

![ディレクトリ作成](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/97d5a1e9-5dfb-4a5c-7374-0d797a92d4b3.png)

#### 2. `Dockerfile` の作成

Ruby のベースイメージを指定した `Dockerfile` を作成します。

```bash
touch Dockerfile
code .
```

```Dockerfile
FROM ruby:3.2.3
WORKDIR /app
```

![Dockerfile作成](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/b5056ee5-8f11-6ffe-5a3f-b47c79894b4e.png)

#### 3. `docker-compose.yml` の作成

Docker Compose を使って開発環境を立ち上げる設定ファイルを作成します。

```bash
touch compose.yml
```

```yaml
services:
  app:
    build: .
    volumes:
      - .:/app
```

![compose.yml作成](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/3991df61-1c0a-c4a1-a9ac-a3027656badf.png)

#### 4. Docker コンテナのビルド

まず **Docker Desktop** を起動しておいてください。

> 🛑 Docker Desktop が起動していないと、以下の操作でエラーになります。

```bash
docker compose build
```

![ビルド](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/eed6b54f-6c61-debc-5378-bb15dfc1a503.png)

## 3. Ruby を実行してみよう

#### 1. コンテナ内に入る

```bash
docker compose run --rm app bash
```

![bashに入る](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/be7acee8-251d-d98c-7dfb-5237a4944bbe.png)

#### 2. Ruby コマンドを実行

```bash
ruby -e "puts 'hello world'"
```

![hello world](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/bc049c2e-8ab2-347c-a7fd-8099ce9ffc96.png)

#### 3. コンテナから抜ける

```bash
exit
```

## まとめ

WSL 2 + Docker を活用することで、Windows 環境でも高速で安定した Ruby 開発が可能になります。

- ✅ WSL 2 により軽量な Linux 実行環境を構築
- ✅ Docker によって Ruby 環境の再現性と分離性を確保
- ✅ Windows の VSCode からスムーズに開発可能

Ruby の学習・開発を Windows 上で始めたい方は、ぜひこの構成を試してみてください！
