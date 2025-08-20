## 「Power Automate × ローカルスクリプト」を無料で双方向連携する方法

## はじめに

業務自動化において、**Power Automate とローカルスクリプトの連携**は非常に強力な手段です。しかし、これらの連携にはしばしば\*\*プレミアムライセンスが必要なコネクター（例：HTTP、Request/Response）\*\*が要求され、導入コストや制約の壁となることがあります。

そこで本記事では、**プレミアムコネクターを一切使わずに、Power Automate とローカルスクリプトを双方向で連携する方法**を、具体的なユースケースと実装例とともに紹介します。

**🎯 ゴール**

以下のような構成を、**すべて無料ライセンスで実現**することを目指します：

| 処理                        | 従来の方法（有料）  | 本記事での代替方法（無料）   |
| --------------------------- | ------------------- | ---------------------------- |
| Power Automate への外部入力 | Request コネクター  | Microsoft Forms + SharePoint |
| 外部 API 呼び出し           | HTTP コネクター     | Node.js スクリプト           |
| Power Automate からの出力   | Response コネクター | SharePoint 経由のポーリング  |

**🛠 想定環境**

- Power Automate（クラウドフロー）
- SharePoint（無料枠）
- Microsoft Graph API（アプリ登録済み）
- Node.js スクリプト（ロカール用）
- Box（ファイル共有、Box API 未使用）
- Viber API（通知）

**🔐 注意点と補足**

- **Graph API 連携にはアプリ登録とトークン管理が必要です**
- **Viber の通知には事前にチャンネルと認証設定が必要です**
- フローの設計とスクリプト側の監視を工夫すれば、**完全無料で高度な連携**が可能です

## スクリプトから Power Automate への通信

Box リンク取得：Box に保存されたファイルの共有リンクを、Power Automate を使って取得し、スクリプトに返す仕組みを構築しました。

#### 1. スクリプト：フォーム送信（Playwright）

```typescript:Playwright.ts
public async submitForm(url: string) {
  const browser = await chromium.launch({ headless: false });
  const page = await browser.newPage();

  await page.goto(url); // 事前登録URLを使用、BoxパスとリクエストIDを送信
  await page.getByRole("button", { name: "送信" }).click();
  await browser.close();
}
```

#### 2. Power Automate：リンク生成

- フォーム送信をトリガーにクラウドフローを開始
- `Box Connector`（無料）を使い、パスからファイル ID 取得
- ファイル ID から共有 🔗 を生成
- `SharePointリスト`に、リクエスト ID をキーに Box リンクを格納

#### 3. スクリプト：定期的にポーリング（Graph API）

```typescript:Graph.ts
public async requestBoxLink(path: string) {
  const uuid = uuidv4();
  const url = [
    `https://forms.office.com/Pages/ResponsePage.aspx?`,
    `id=<フォームID>`,
    `&<パスのフィールドID>=${path}`,
    `&<リクエストIDのフィールドID>=${uuid}`,
  ].join("");

  await Playwright.submitForm(url);
  return uuid;
}

public async getSharedLinkResponse(requestId: string) {
  const siteId = "xxx"; // sharepointサイトのID
  const list = "xxx"; // リストID
  const url = "https://graph.microsoft.com/v1.0/sites/${siteId}/lists/${listId}/items";
  const params = {
    $expand: "fields",
    $filter: `fields/requestId eq '${requestId}'`,
    $orderby: "createdDateTime desc",
    $top: 1,
  };

  const res = await Postman.getWithParams(url, params, this.accessToken); // axiosのラッパー
  const record = res.data.value.pop();
  record ? return record.files.data : return "";
}
```

#### 4. スクリプト：リンク取得完了まで待機

- Box リンクが取得できたら処理を終了
- 取得できなければ数秒後に再試行（例：2 秒ごとに最大 30 回）

```typescript:FileUtil.ts
public async getSharedLink(boxPath: string) {
  const requestId = await Graph.requestBoxLink(boxPath);
  let sharedLink = "";
  while (!sharedLink) {
    await CommonUtil.wait(2000);
    sharedLink = await Graph.getSharedLinkResponse(requestId);
  }

  return sharedLink;
}
```

## Power Automate からスクリプトへの通信

Power Automate から生成された通知情報を、スクリプトが定期的に取得し、Viber へ一括送信する仕組みです。

#### 1. Power Automate：通知を SharePoint に格納

- 通知が発生したら、SharePoint リストに 1 レコード追加
  - 通知内容（メッセージ、宛先、認証トークンなど）を SharePoint リストに記録
  - タスク名：`send-viber`

#### 2. スクリプト：1 時間ごとにリストをチェック

```typescript:Graph.ts
public async getViberTasks() {
  const siteId = "xxx"; // sharepointサイトのID
  const list = "xxx"; // リストID
  const url = "https://graph.microsoft.com/v1.0/sites/${siteId}/lists/${listId}/items";
  const params = {
    $expand: "fields",
    $filter: `fields/taskName eq '$send-viber'`,
    $orderby: "createdDateTime desc",
  };

  type Record = {
    id: string;
    fields: {
      taskName: string;
      data: string;
      requestId: string;
      authToken: string;
      broadcaster: string;
    };
  };

  const res = await Postman.getWithParams(url, params, this.accessToken); // axiosのラッパー
  const records = res.data.value as Record[];

  if (records.lenght > 0) {
    const ids = records.map((record) => record.id);
    const tasks = records.map((record) => ({
      authToken: record.fields.authToken,
      broadcaster: record.fields.broadcaster,
      msg: record.fields.data,
    }));

    await Promise.all(ids.map((id: string) => this.deleteItem(id)));
    return tasks;
  } else {
    return [];
  }
}
```

#### 3. スクリプト：1 時間ごとに Viber へ通知

```ts:Viber.ts
public async sendQueuedMessages() {
  const tasks = await Graph.getViberTasks();
  for (const task of tasks) {
    await this.postToChannel(task.authToken, task.broadcaster, task.msg);
  }
}
```

```ts:TaskScheduler.ts
private taskConfig: TaskConfig[] = [
  {
    name: "sendQueuedMessages",
    cronExpression: "0 * * * *",
    command: async () => {
      await Viber.sendQueuedMessages();
    },
  }
];
```

## まとめ

本記事では、**Power Automate の無料枠を最大限に活用し、ローカルスクリプトとの双方向連携を実現する手法**を紹介しました。

| 通信方向        | 使用技術                     | プレミアム不要 |
| --------------- | ---------------------------- | -------------- |
| スクリプト → PA | Microsoft Forms + SharePoint | ✅             |
| PA → スクリプト | SharePoint ポーリング        | ✅             |

Power Automate の制限に悩んでいる方、プレミアム契約なしで業務自動化を進めたい方にとって、参考になる構成になっているはずです。ぜひお試しください！
