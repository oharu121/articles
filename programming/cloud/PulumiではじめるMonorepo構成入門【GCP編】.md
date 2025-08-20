## はじめに 

「インフラのコード（IaC）を、アプリケーションのコードと一緒に管理したい。でも、具体的にどこに何を書けばいいのか分からない…」

そんな悩みを持つ開発者のあなたへ。本記事では、Pulumi × Monorepo構成で始める、シンプルかつ実践的なインフラ管理の方法を、Google Cloud Platform (GCP) への自動デプロイ例と共にハンズオン形式で解説します。

この記事を読み終える頃には、以下のスキルが身についています。

>* Node.jsアプリケーションとインフラコード（Pulumi）を1つのリポジトリで管理する方法
>* GCP上にコンテナをデプロイするためのインフラ（Artifact Registry, Cloud Run）をコード化するスキル
>* Pull Requestをトリガーにデプロイプレビューを行い、mainブランチへのマージで本番環境へ自動デプロイするCI/CDパイプラインの構築方法

今回は、パブリックアクセス可能なCloud RunサービスをGCPに構築する簡単な例を通して、Pulumiの導入方法とMonorepoのベストプラクティスを学びましょう。

## Monorepo構成とは？

Monorepo（モノレポ）とは、**アプリとインフラのコードを1つのリポジトリにまとめる構成**です。

```text:フォルダ構成
my-project/
├── infrastructure/              # IaC (Pulumi) コードを格納
│   ├── Pulumi.dev.yaml          # 開発環境用の設定
│   ├── Pulumi.staging.yaml      # ステージング環境用の設定
│   ├── Pulumi.production.yaml   # 本番環境用の設定
│   └── index.ts                 # 環境共通のインフラ定義
├── app/                         # アプリケーションコードを格納
│   ├── index.js
│   └── Dockerfile
├── package.json                 # プロジェクト全体の依存関係など
└── .github/workflows/           # CI/CD (GitHub Actions) の定義
```

**📌 ポイント**

>* **`infrastructure/`**: 各環境（`dev`, `staging`, `production`）に応じた設定ファイルを分けて、`index.ts` に共通のPulumi定義を記述
>* **`app/`**: アプリケーションのコードと `Dockerfile` を配置
>* **`.github/workflows/`**:  CI/CDの定義を配置します。Monorepoでは、特定のディレクトリ（例: `app/` や `infrastructure/`）の変更を検知して、関連するジョブのみを実行することで、パイプラインを効率化できる

## なぜPulumiを使う？

gcloud CLI（コマンド）で手動デプロイも可能ですが、なぜIaCツール、特にPulumiを使うのでしょうか？

Pulumiは **TypeScriptやPythonといった汎用プログラミング言語**でインフラを構築できるため、アプリケーション開発者にとって学習コストが低く、多くのメリットがあります

| 比較項目      | Pulumi                            | gcloud CLI            |
| --------- | --------------------------------- | --------------------- |
| **再利用性**  | 関数やクラスでコードを部品化・共通化でき、高品質なインフラを効率的に構築できる。              | シェルスクリプトでの再利用が中心となり、複雑化しやすい。      |
| **可読性・レビュー**   | 宣言的に「あるべき状態」を記述。Gitで差分が明確になり、コードレビューが容易。 | 手順（コマンドの羅列）を記述するため、最終的な状態を把握しにくい。   |
| **状態管理**   | `pulumi up` を実行する前に、**変更内容をプレビュー（`pulumi preview`）**できる。 | コマンド実行結果を都度確認する必要があり、意図しない変更のリスクがある。   |
| **環境分離**  | 	**スタック（Stack）**という機能で、開発・本番環境の設定を完全に分離・管理できる。                 | プロジェクトIDや設定値をコマンドの引数や環境変数で切り替える必要があり、ミスを誘発しやすい。 |

## Pulumiにおける「スタック」とは？

Pulumiでは、環境ごとの設定やデプロイされたリソースの状態を「**スタック（Stack）**」という単位で管理します。これにより、同じインフラ定義コードを使いながら、環境ごとに異なる構成を安全に展開できます。
| スタック名        | 用途例      | 備考                |
| ------------ | -------- | ----------------- |
| `dev`        | 開発環境     | 個人の開発やPull Requestごとの一時的なテスト環境用。低スペック・低コストなリソースを選ぶ。         |
| `staging`    | ステージング環境 | 本番環境とほぼ同等の構成を持つ検証環境。QAチームのテストやリリース前の最終確認に利用。        |
| `production` | 本番環境     | エンドユーザーに提供するサービス。安定性とセキュリティを最優先し、監視や承認フローを厳格にする。 |

スタックごとに `Pulumi.<stack-name>.yaml` を用意し、プロジェクトIDやGCPリージョン、バケット名などの設定値を個別に定義できます。

```bash
# 'dev' という名前のスタックを作成
pulumi stack init dev

# 現在のスタック (dev) に設定値を追加
pulumi config set gcp:project my-dev-project
pulumi config set gcp:region asia-northeast1
```

こうすることで、同じ`index.ts`でもスタックに応じて異なるリソースが展開されます。

## 1\. プロジェクトのセットアップ

まず、作業ディレクトリを作成し、簡単なWebアプリケーションとDockerfileを用意します。

```bash
# プロジェクトルートを作成
mkdir my-project && cd my-project

# アプリケーションディレクトリを作成
mkdir app && cd app

# Node.jsプロジェクトを初期化し、Expressをインストール
npm init -y
npm install express
```

`app/index.js` を作成します。

```js:app/index.js
const express = require("express");
const app = express();
const port = process.env.PORT || 8080;

app.get("/", (req, res) => {
  res.send("Hello from Pulumi-managed Cloud Run!");
});

app.listen(port, () => {
  console.log(`Listening on port ${port}`);
});
```

次に、`app/Dockerfile` を作成します。

```dockerfile:app/Dockerfile
FROM node:18

WORKDIR /app
COPY package*.json ./
RUN npm install --only=production
COPY . .

ENV PORT=8080
CMD ["node", "index.js"]
```

## 2\. Pulumiプロジェクトの初期化

次に、インフラ用のディレクトリを作成し、Pulumiプロジェクトを初期化します。

```bash
# プロジェクトルートに戻る
cd ..

# インフラ用ディレクトリを作成
mkdir infrastructure && cd infrastructure

# Pulumiプロジェクトを新規作成 (TypeScriptを選択)
pulumi new typescript
````

**📌 ポイント**

>* `pulumi new` を実行すると、プロジェクト名や説明などを対話形式で聞かれる
>* 入力後、`Pulumi.yaml` と `index.ts` などが生成される
>* `Pulumi.yaml` がインフラプロジェクトの定義ファイル

`Pulumi.yaml`は以下のように編集しておきましょう。

```yaml:Pulumi.yaml
name: my-cloudrun-infra
runtime: nodejs
description: A Pulumi project to deploy a Cloud Run service on GCP
```

必要なPulumiのGCPプラグインとDockerプラグインをインストールします。

```bash
# @pulumi/gcp: GCPリソースを操作
# @pulumi/docker: Dockerイメージのビルドとプッシュを操作
npm install @pulumi/gcp @pulumi/docker
```

## 3\. GCPの認証とAPIの有効化

PulumiがGCPリソースを操作できるように、ローカル環境で認証を通し、必要なAPIを有効化します。

```bash
# GCPアカウントにログイン
gcloud auth login

# アプリケーションのデフォルト認証情報を設定
gcloud auth application-default login

# 操作対象のGCPプロジェクトを設定 (your-project-id は実際のIDに置き換える)
gcloud config set project your-project-id

# 必要なGCPサービスAPIを有効化
gcloud services enable \
  iam.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com
```

## 4\. Iacコードの作成

いよいよインフラをコードで定義します。`infrastructure/index.ts` を以下の内容に書き換えてください。

このコードは、次の3つの処理を行います。

> 1\. **Artifact Registryリポジトリの作成**: Dockerイメージを保存する場所を作成します。
> 2\. **Dockerイメージのビルド＆プッシュ**: app ディレクトリのコードをビルドし、先ほど作成したリポジトリにプッシュします。
> 3\. **Cloud Runサービスの作成**: プッシュしたイメージを使ってCloud Runサービスをデプロイし、誰でもアクセスできるように設定します。

```ts
import * as pulumi from "@pulumi/pulumi";
import * as gcp from "@pulumi/gcp";
import * as docker from "@pulumi/docker";

// Pulumiの設定ファイルから値を取得
const config = new pulumi.Config();
const gcpConfig = new pulumi.Config("gcp");
const location = gcpConfig.require("region"); // 例: "asia-northeast1"
const project = gcpConfig.require("project");
const appName = config.get("appName") || "my-app";

// 1. Dockerイメージを保存するArtifact Registryリポジトリを作成
const repository = new gcp.artifactregistry.Repository("my-repo", {
    location: location,
    repositoryId: `${appName}-repo`,
    format: "DOCKER",
    description: "Docker repository for my-app",
});

// リポジトリURLを動的に生成
const repositoryUrl = pulumi.interpolate`${location}-docker.pkg.dev/${project}/${repository.repositoryId}`;

// 2. ローカルのappディレクトリからDockerイメージをビルドし、Artifact Registryにプッシュ
//    `pulumi up` を実行するマシンでDockerが実行されている必要があります。
const image = new docker.Image(appName, {
    imageName: pulumi.interpolate`${repositoryUrl}/${appName}:latest`,
    build: {
        context: "../app", // Dockerfileがあるディレクトリのパス
        platform: "linux/amd64", // M1/M2 MacなどでもCloud Runで動くように指定
    },
});

// 3. Cloud Runサービスを作成
const service = new gcp.cloudrun.Service(appName, {
    location: location,
    template: {
        spec: {
            containers: [{
                image: image.imageName, // ビルド&プッシュしたイメージ名を指定
            }],
        },
    },
});

// 4. Cloud Runサービスを一般公開（allUsers）に設定
const iam = new gcp.cloudrun.IamMember(`${appName}-invoker`, {
    service: service.name,
    location: service.location,
    role: "roles/run.invoker",
    member: "allUsers",
});

// デプロイ完了後に出力するURL
export const url = service.statuses[0].url;
```

## 5\. スタックの作成とデプロイ

コードが完成したので、dev 環境用のスタックを作成し、デプロイします。

```bash
# infraディレクトリにいることを確認
# pulumi loginコマンドでPulumiにログインしておくこと（初回のみ）

# 'dev'スタックを初期化
pulumi stack init dev

# 'dev'スタックに設定値をセット (your-project-idは自分のものに)
pulumi config set gcp:project your-project-id
pulumi config set gcp:region asia-northeast1
pulumi config set appName my-app-dev

# プレビューで変更内容を確認
pulumi preview

# 確認後、デプロイを実行
pulumi up
```

**✅ 実行結果**

成功すると、コンソールにCloud RunのURLが出力されます。

```
Outputs:
    url: "https://my-app-dev-xxxxxxxxx-an.a.run.app"

```

>上記URLにアクセスすれば、Hello from Cloud Run! が表示されます 🎉

## 6\. GitHub Actionsでデプロイを自動化

最後に、このプロセスをCI/CDで自動化しましょう。Pull Request作成時に`pulumi preview`を、`main`ブランチへのマージ時に`pulumi up`を実行するワークフローを作成します。

まず、GitHubリポジトリに以下の2つのSecretを設定します。

>* `PULUMI_ACCESS_TOKEN`: [PulumiのAccess Token](https://www.pulumi.com/docs/pulumi-cloud/access-management/access-tokens/) ページで生成します。
>* `GCP_SA_KEY`: GCPの [サービスアカウントキー](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) をJSON形式で作成し、その中身をSecretに設定します。サービスアカウントには「Artifact Registry管理者」「Cloud Run管理者」「サービスアカウントユーザー」などのロールを付与してください。

次に、`.github/workflows/pulumi.yml` をプロジェクトルートに作成します。

```yaml:.github/workflows/pulumi-preview.yml
name: Pulumi CI/CD

on:
  pull_request:
    branches:
      - main
    paths:
      - 'app/**'
      - 'infrastructure/**'
  push:
    branches:
      - main
    paths:
      - 'app/**'
      - 'infrastructure/**'

jobs:
  pulumi:
    name: Pulumi Up/Preview
    runs-on: ubuntu-latest
    
    # ワークフロー内でGCPに認証するための設定
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # GCPに認証 (Workload Identity連携)
      - name: Authenticate to Google Cloud
        uses: 'google-github-actions/auth@v2'
        with:
          workload_identity_provider: 'projects/YOUR_GCP_PROJECT_NUMBER/locations/global/workloadIdentityPools/YOUR_POOL_ID/providers/YOUR_PROVIDER_ID'
          service_account: 'your-service-account-email@your-project-id.iam.gserviceaccount.com'

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Setup Pulumi
        uses: pulumi/actions@v5
        with:
          pulumi-version: 'latest'

      - name: Install Dependencies
        run: npm install
        working-directory: ./infrastructure

      # Pull Requestの場合は preview を実行
      - name: Pulumi Preview
        if: github.event_name == 'pull_request'
        run: pulumi preview --stack staging
        working-directory: ./infrastructure
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}

      # mainブランチへのPushの場合は up を実行
      - name: Pulumi Up
        if: github.event_name == 'push'
        run: pulumi up --yes --stack staging
        working-directory: ./infrastructure
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
```

| トリガーイベント                    | 実行内容                     | 目的・備考                                                 |
| --------------------------- | ------------------------ | ----------------------------------------------------- |
| Pull Request作成時             | `pulumi preview` を実行     | 変更によってインフラがどう変わるかを事前に確認し、意図しない破壊的な変更を防ぐ。レビューの品質向上。    |
| mainブランチへのマージ時              | `pulumi up` を実行          | staging環境など、検証用の環境へ自動で変更を反映させる。                       |
| 手動実行時 (`workflow_dispatch`) | 承認フロー付きで `pulumi up` を実行 | production環境へのデプロイは、Slack通知やGitHub上での手動承認を経て、安全に実行する。 |

## おわりに

本記事では、PulumiとMonorepo構成を使って、GCP上にCloud Runサービスを構築・自動デプロイする一連の流れを解説しました。

この構成のメリットを改めてまとめます。

>* **統一された管理**: アプリとインフラを1つのリポジトリでバージョン管理できる。
>* **高い生産性**: TypeScriptのような慣れた言語でインフラを記述でき、コンポーネント化による再利用も容易。
>* **安全で効率的な運用**: pulumi previewによる事前確認と、GitHub Actionsによる自動化で、安全かつ効率的にインフラを更新できる。

まずはこのシンプルな構成から、あなたのプロジェクトにモダンなインフラ管理を取り入れてみてください。
