## jsforce で Salesforce レポートを取得・分析する方法

## はじめに

Salesforce に保存されたレポートやデータを、Node.js から自動取得したいケースは多いです。この記事では、[jsforce](https://github.com/jsforce/jsforce) ライブラリを使って以下の内容を解説します。

- レポートの構成情報（カラム・集計など）を取得する方法
- レポートを直接実行してデータを抽出する方法
- より自由度の高い SOQL クエリによるレポート抽出方法（クラス形式で紹介）

## 事前準備：Salesforce アカウントのセットアップ

1. Salesforce から届く「Salesforce へようこそ」メールの「アカウントを確認」をクリック
2. 「パスワードのリセット」を実行し、パスワードと秘密の質問を設定
3. メール記載のユーザー名と ② で設定したパスワードでログイン
4. 端末に「Salesforce Authenticator」アプリをインストールし、連携設定
5. 今後は、ユーザー名＋パスワード＋モバイル認証でログインが可能になります
6. `jsforce`ライブラリの導入

```bash
npm install jsforce dayjs
```

`jsforce`は Salesforce の REST API クライアントで、SOQL 実行・レポート取得・オブジェクト操作などが可能です。

## レポート構成の確認

Salesforce のレポートは ID で指定できます。
まずは`getReportInfo()`を使って、レポートの**構成・集計項目・使用カラム**などのメタデータを確認しましょう。

```ts
import jsforce from "jsforce";

const conn = new jsforce.Connection({
  loginUrl: "https://login.salesforce.com",
});
await conn.login("your-username@example.com", "your-password");

const reportId = "00O5g000005AbCd"; // 任意のレポートID
const reportSchema = await conn.analytics.report(reportId).describe();

console.log(JSON.stringify(reportSchema, null, 2));
```

📂 この結果には、次のような情報が含まれます：

- 使用カラム（`detailColumns`）
- 絞り込み条件
- グルーピング設定（groupingsDown, groupingsAcross）
- 使用されている集計関数や数式項目

## 方法 1：レポートを直接実行（非推奨）

構成がわかったら、`execute()` を使ってレポートを実行しデータを取得できます。

```ts
const schema = await conn.analytics.report("00O5g000005AbCd");
const execResult = await schema.execute({ details: true });

const rows: object[] = [];

for (const key in execResult.factMap) {
  const fact = execResult.factMap[key];
  if (fact?.rows) {
    fact.rows.forEach((row) => {
      const rowData: { [key: string]: string } = {};
      execResult.reportMetadata.detailColumns.forEach((col, index) => {
        rowData[col] = row.dataCells[index]?.label || "";
      });
      rows.push(rowData);
    });
  }
}

console.log(rows);
```

:::note warn
Salesforce レポートの `execute()` は**実行回数に制限**があります。大量データや定期処理には **SOQL クエリの方が安全** です。
:::

## 方法 2：SOQL クエリでデータ抽出（推奨）

#### 🧭 想定シナリオ

> 製造業の Salesforce 環境において、「製品リクエスト（Product_Request\_\_c）」オブジェクトから、**今後 1 か月以内に納品予定の製品リクエスト**を抽出し、**数量と希望納品日ごとに並べて出力したい**。

#### クラス定義とクエリメソッド

```ts:Salesforce.ts
import jsforce from "jsforce";
import dayjs from "dayjs";
import credentials from "@utils/constants/credentials";
import apis from "@utils/constants/apis";

class Salesforce {
  private username = credentials.SALESFORCE_USERNAME;
  private password = credentials.SALESFORCE_PASSWORD;
  private isLogin = false;

  private client = new jsforce.Connection({
    loginUrl: apis.SALESFORCE_ENDPOINT,
  });

  private async checkConnection(): Promise<void> {
    if (!this.isLogin) await this.connect();
  }

  private async connect() {
    await this.client.login(this.username, this.password);
    this.isLogin = true;
  }

  /**
   * 製品リクエスト（カスタムオブジェクト）から今後1か月分の出荷予定を抽出
   */
  public async getUpcomingProductRequests() {
    await this.checkConnection();

    const today = dayjs().startOf("day").toISOString();
    const oneMonthLater = dayjs().add(30, "day").endOf("day").toISOString();

    const result = await this.client
      .sobject("Product_Request__c")
      .select([
        "Product__r.Name",                // 製品名（参照関係）
        "Requested_Quantity__c",         // 希望数量
        "Preferred_Ship_Date__c",        // 希望納品日
        "Request_Status__c",             // ステータス（例: 承認済み）
      ])
      .where({
        Request_Status__c: "承認済み",
        Preferred_Ship_Date__c: {
          $gte: jsforce.Date.toDateLiteral(today),
          $lte: jsforce.Date.toDateLiteral(oneMonthLater),
        },
      })
      .orderby("Preferred_Ship_Date__c", "ASC")
      .execute({ autoFetch: true });

    return result;
  }
}

export default new Salesforce();
```

💡 主なポイント

| 項目                     | 説明                                                      |
| ------------------------ | --------------------------------------------------------- |
| `Product_Request__c`     | 製品リクエストを記録するカスタムオブジェクト              |
| `Product__r.Name`        | 参照先の製品オブジェクトから製品名を取得                  |
| `Preferred_Ship_Date__c` | 納品希望日で範囲を絞る                                    |
| `Request_Status__c`      | ステータスが「承認済み」のみに限定                        |
| 日付条件                 | `jsforce.Date.toDateLiteral()` で SOQL 形式へ変換         |
| クエリ結果               | 自動ページング（`autoFetch: true`）で最大件数の制限に対応 |

#### 📋 出力例（例: コンソール表示）

```json
[
  {
    "Product__r": { "Name": "工業用バルブ" },
    "Requested_Quantity__c": 50,
    "Preferred_Ship_Date__c": "2024-06-01",
    "Request_Status__c": "承認済み"
  },
  {
    "Product__r": { "Name": "制御装置A" },
    "Requested_Quantity__c": 20,
    "Preferred_Ship_Date__c": "2024-06-05",
    "Request_Status__c": "承認済み"
  }
]
```

#### ✅ 使用例

```ts
import Salesforce from "./Salesforce";

(async () => {
  const requests = await Salesforce.getUpcomingProductRequests();
  console.table(
    requests.map((r) => ({
      製品名: r.Product__r?.Name,
      数量: r.Requested_Quantity__c,
      納品希望日: r.Preferred_Ship_Date__c,
    }))
  );
})();
```

## まとめ

| 目的                 | 実装内容                                           |
| -------------------- | -------------------------------------------------- |
| 製品リクエストの抽出 | カスタムオブジェクト + フィルタ条件による絞り込み  |
| 日付範囲             | `dayjs + jsforce.Date.toDateLiteral()`で柔軟に指定 |
| 安全性               | 本番環境の項目名・用語は使用せず、マスキング済み   |

次回は、このデータを **CSV 出力** や **Google Sheets 連携**する方法も紹介予定です。
この記事が役立ったら「いいね」やフォローをお願いします！質問もお気軽にどうぞ。
