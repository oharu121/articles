# KintoneのSDKで業務レポートを自動出力する方法

# はじめに 

業務でKintoneを使っていて、「条件に合うレコードだけ抽出してCSV出力したい」と思ったことはありませんか？この記事では、[公式のKintone REST API Client SDK](https://github.com/kintone/js-sdk/tree/master/packages/rest-api-client) を使って、Kintoneからデータを取得し、整形する方法をご紹介します。

# 🛠️ 事前準備

### 1. APIトークンを取得する

1. Kintoneにログイン
2. 対象アプリを開く
3. 「設定」→「APIトークン」へ移動
4. 必要な権限（例：レコード閲覧）を付与して生成
5. トークンをコピーして、`.env`や設定ファイルに保存

### 2. アプリIDを確認する

アプリのURL末尾にある数字がアプリIDです（例：`https://{yourdomain}.cybozu.com/k/{アプリID}/`）

下記の記事を参考しました：

https://zenn.dev/ak2ie/articles/9dff54b6b2c240

### 3. SDKのセットアップ

以下のように `@kintone/rest-api-client` をインストールします。

```bash
npm install @kintone/rest-api-client
```

環境変数などにトークンやURLを記載し、安全に管理します。

# アプリの一覧設定の取得（Postmanなど）

Kintoneには、GUI上で設定された「一覧（ビュー）」が存在します。
**この一覧の設定内容を事前にAPIから取得しておくことで、どんなクエリやソート条件が使われているかを把握できます。**

たとえば、「一覧1」にはどんな絞り込み条件や表示フィールドがあるのかを確認して、それをプログラムに活用することが可能です。

**エンドポイント（通常アプリの場合）**

```
GET https://{yourdomain}.cybozu.com/k/v1/app/views.json
```

**ヘッダー**

```json
{
  "X-Cybozu-API-Token": "API_TOKEN",
  "Content-Type": "application/json"
}
```
* API_TOKEN：取得したAPIトークン

**リクエストボディの例**

```json
{
  "app": 123,
  "lang": "ja"
}
```
* app：取得したアプリID

レスポンス例（抜粋）

```json
{
  "views": {
    "一覧1": {
      "type": "LIST",
      "name": "一覧1",
      "fields": ["レコード番号", "文字列1行_0"],
      "filterCond": "更新日時 > \"2024-05-01T00:00:00Z\"",
      "sort": "レコード番号 asc"
    }
  },
  "revision": "1"
}
```

💡 ここで得られる情報の使いどころ

* `filterCond` … クエリ構文の参考にできる（コードで再現する際に便利）
* `fields` … 表示されるフィールドの確認（マッピングに役立つ）
* `sort` … 並び順の参考

# レポート生成クラス実装

以下のコードは、条件に合致したKintoneレコードを一括取得し、必要な情報だけをマッピングした配列を生成するものです。

```ts:Kintone.ts
import { KintoneRestAPIClient } from "@kintone/rest-api-client";
import credentials from "@utils/constants/credentials";

class Kintone {
  private appId = "123"; // 使用するKintoneアプリID
  public client = new KintoneRestAPIClient({
    baseUrl: "https://{yourdomain}.cybozu.com", // あなたのドメインに変更
    auth: { apiToken: credentials.KINTONE_API_TOKEN },
  });

  /**
   * GUIで定義された一覧（ビュー）の設定を取得する
   */
  public async getViewSettings() {
    const result = await this.client.app.getViews({ app: this.appId, lang: "ja" });

    console.log("=== 一覧設定 ===");
    for (const view of Object.values(result.views)) {
      console.log(`📄 一覧名: ${view.name}`);
      console.log(`🔍 フィルター条件: ${view.filterCond}`);
      console.log(`📑 フィールド: ${view.fields?.join(", ")}`);
      console.log(`🔃 ソート: ${view.sort}`);
      console.log("-----------------------------");
    }

    return result.views;
  }

  /**
   * フィールドコードをラベルにマッピングしてレポートを生成
   */
  public async getReportData() {
    // フィールドコード ➝ ラベル名のマッピング
    const translations: { [fieldCode: string]: string } = {
      レコード番号: "レコード番号",
      文字列1行_0: "顧客名",
      文字列1行_1: "電話番号",
      数値_0: "注文数",
      日付_0: "注文日",
    };

    const query = `注文ステータス in ("完了") and 日付_0 >= "2024-01-01"`; // 任意の絞り込み

    const cursor = await this.client.record.createCursor({
      app: this.appId,
      fields: Object.keys(translations),
      query,
      size: 500,
    });

    const result: any[] = [];
    let hasNext = true;

    while (hasNext) {
      const res = await this.client.record.getRecordsByCursor({ id: cursor.id });
      result.push(...res.records);
      hasNext = res.next;
    }

    // ラベル付きのデータに変換
    const remapped = result.map((record) => {
      const row: { [label: string]: string } = {};

      for (const [fieldCode, label] of Object.entries(translations)) {
        const raw = record[fieldCode]?.value;
        row[label] = typeof raw === "string" ? raw : raw != null ? String(raw) : "";
      }

      return row;
    });

    return remapped;
  }
}

export default new Kintone();
```

# 実行例

```ts
const kintone = new Kintone();

(async () => {
  await kintone.getViewSettings(); // 一覧設定の確認（開発補助）

  const data = await kintone.getReportData(); // レポートデータ取得
  console.table(data);
})();
```

### 🧾 出力イメージ（例）

```ts
[
  { 顧客名: "田中株式会社", 電話番号: "03-1234-5678", 注文数: "10", 注文日: "2024-05-01" },
  { 顧客名: "鈴木商事", 電話番号: "06-9876-5432", 注文数: "5", 注文日: "2024-05-02" },
]
```

# 補足：なぜマッピングが必要なのか？

Kintoneのレコードデータは「**フィールドコード**」をキーに返ってきます。
しかし、**GUI上に表示されるラベル（項目名）とは異なる**ことがほとんどです。

たとえば、GUIで「顧客名」と表示されているフィールドのコードは `文字列1行_0` という機械的な名前かもしれません。

| フィールドコード  | 表示ラベル（例） |
| --------- | -------- |
| `文字列1行_0` | 顧客名      |
| `文字列1行_1` | 電話番号     |
| `数値_0`    | 注文数      |

そのため、コード上では下記のように**対応表（マッピング）を定義**しておくと、
出力時に「わかりやすい形」でデータを扱うことができます。

```ts
const translations = {
  文字列1行_0: "顧客名",
  文字列1行_1: "電話番号",
  数値_0: "注文数",
};
```

# 応用例

* **CSVとしてファイル出力する**
* **Google Sheetsに連携する**
* **レコードの一括更新（`updateRecords`）**
* **定期実行（cron + Cloud Functions）**

# まとめ

KintoneのREST API SDKを使えば、GUIからでは難しい細かいデータ抽出や業務自動化が可能です。特にサブテーブルの取り扱いや条件指定は、レポート業務で重宝します。

社内業務の効率化や自動レポート化の第一歩として、ぜひ取り入れてみてください！
