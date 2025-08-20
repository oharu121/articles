## はじめに

「Infrastructure as Code（IaC、インフラストラクチャ・アズ・コード）」とは、**インフラ（サーバー、ネットワーク、DBなど）をコードで定義・管理する手法**です。

**✅ 主なIaCツールの比較**

| ツール名                   | 特徴                               |
| ---------------------- | -------------------------------- |
| **Terraform**          | マルチクラウド対応、宣言型、業界で広く使われている        |
| **AWS CloudFormation** | AWS専用、公式サポート、やや複雑                |
| **Ansible**            | 手続き型（Pythonベース）、設定変更やプロビジョニングに強い |
| **Pulumi**             | コードベース（TypeScriptなど）、開発者向け       |


**✅ 主なアプローチの比較**

| 方式                   | 説明                             | 代表例                       |
| -------------------- | ------------------------------ | ------------------------- |
| **宣言型（Declarative）** | 「**こうなっていてほしい最終的な状態** 」をコードに記述する方式。 | Terraform, CloudFormation |
| **手続き型（Imperative）** | 「**どのような手順で構成するか**」を明示的に記述する方式。    | Ansible, Chef             |

https://qiita.com/oharu121/items/7d3ba5c1a691ceeaf417

## なぜIaCが重要なのか？

**❌ 従来のやり方（手動構築）の問題点**

>* 設定ミスが起きやすい
>* 環境ごとに違いが出てしまう（開発・ステージング・本番）
>* ドキュメントが現実とズレがち
>* 構築に時間がかかる

**✅ IaCのメリット**

>* コードで構成管理 → **Gitでバージョン管理できる**
>* 再現性のある環境を即座に構築できる（**DevOps・CI/CD**に必須）
>* インフラのレビューやテストも可能になる
>* マルチクラウドやマルチ環境への対応も簡単

## 1\. 宣言型（Declarative）

**✅ 特徴**

>* **目的地を指定する**：最終的にどんな状態になっていればよいかを記述
>* **状態の自動管理**：ツールが現状との差分を検出し、必要な変更だけを適用
>* **再現性が高い**：意図した状態に何度でも収束できる

**🧰 代表ツール**

>* **Terraform**：最も人気のある宣言型ツール。マルチクラウド対応。
>* **CloudFormation**：AWS公式のテンプレートベースツール。

**🧰 サンプル（Terraform）**

```hcl
resource "google_storage_bucket" "my_bucket" {
  name          = "my-bucket"
  location      = "ASIA"
  force_destroy = true
}
```

## 2\. 手続き型（Imperative）

**✅ 特徴**

>* **工程を明示する**：「これをして、次にこれをして…」という命令をコード化
>* **柔軟性が高い**：複雑なロジックや分岐処理が書きやすい
>* **インフラの操作ログのように記述可能**

**🧰 代表ツール**

>* **Ansible**：シンプルな構文とエージェントレスで人気
>* **Chef**：Rubyベースの強力な自動化フレームワーク

**🧰 サンプル（Ansible）**

```yaml
- name: Create GCP storage bucket
  hosts: localhost
  tasks:
    - name: Create bucket
      gcp_storage_bucket:
        name: my-bucket
        location: ASIA
        state: present
```

## Pulumiはどっち？

Pulumiは少し特殊です。**「成果物としては宣言型」** でありながら、**「記述方法は手続き型」** というハイブリッド型といえます。

**🧾 Pulumiの特徴**

>* **TypeScript, Python, Go, C# などで記述可能**（コードはImperative風）
>* **実行結果は宣言的に処理される**（差分を検出して状態を管理）
>* **開発者フレンドリー**：ロジックや条件分岐、ループも記述しやすい
>* **IDE補完や型安全**が得られる → Terraformよりデバッグしやすい

**🧰 サンプル（Pulumi / TypeScript）**

```ts
import * as gcp from "@pulumi/gcp";

const bucket = new gcp.storage.Bucket("my-bucket", {
  location: "ASIA",
  forceDestroy: true,
});
```

**🔁 条件分岐・ループが自然に書ける**

```ts
const bucketNames = ["a-bucket", "b-bucket", "c-bucket"];

bucketNames.forEach((name) => {
  new gcp.storage.Bucket(name, {
    location: "ASIA",
    forceDestroy: true,
  });
});
```

**📌 まとめると…**

| 特徴           | Terraform   | Ansible    | Pulumi                    |
| ------------ | ----------- | ---------- | ------------------------- |
| 記述方法   | 宣言型         | 手続き型       | **手続き型のように書く**            |
| 実行原理    | 宣言型（差分適用）   | 手続き型（逐次処理） | **宣言型（差分適用）**             |
| 条件分岐・ループの柔軟性 | 制限あり        | 自由         | **自由（言語機能が使える）**          |
| 適した開発者層      | インフラ専任エンジニア | 運用・構築チーム向け | **アプリ開発者やフルスタックエンジニアにも◎** |

## おわりに

Pulumiは、アプリ開発の延長線でインフラもコードで管理したいというニーズに応える、非常に柔軟なIaCツールです。宣言型と手続き型、両方のいいとこ取りをしたい方に特におすすめです！
