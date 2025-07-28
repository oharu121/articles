# はじめに

**✉️ やりたいこと**

* Microsoft Graph API を使ってメールを送信したい
* テンプレートHTMLを使ってメール本文を整形したい
* 添付ファイル、CC、BCCも柔軟に設定したい

**💡 実装のポイント**

* `fs.readFileSync()` でテンプレートHTMLを読み込み
* テンプレート内の `{{MAIL_CONTEXT}}` にHTML文字列を挿入
* 添付ファイルを Base64 に変換し、Graph API の `attachments` として追加
* `cc`, `bcc` を可変に対応させ、空なら省略する実装に

Microsoft Graph APIラッパークラスの実装は、以下の記事をご参照ください。

https://qiita.com/oharu121/items/b1d18d73f8240c9e4c1b

# 1. メール送信用関数の実装

```ts:Graph.ts
public async sendMail(
  subject: string,
  to: string,
  ccs: string[],
  bccs: string[],
  body: string,
  attachmentPaths: string[] // パスの文字列
) {
  await this.checkToken();

  const url = "https://graph.microsoft.com/v1.0/me/sendMail";

  const attachments = [];
  for (const attachmentPath of attachmentPaths) {
    try {
      const fileContent = fs.readFileSync(attachmentPath);
      const fileName = path.basename(attachmentPath);

      attachments.push({
        "@odata.type": "#microsoft.graph.fileAttachment",
        name: fileName,
        contentType: "application/vnd.ms-excel",
        contentBytes: fileContent.toString("base64"),
      });
    } catch (error) {
      console.error(`Error reading attachment ${attachmentPath}:`, error);
      continue;
    }
  }

  const emailMessage = {
    message: {
      subject,
      body: {
        contentType: "HTML",
        content: body,
      },
      toRecipients: [{ emailAddress: { address: to } }],
      ...(ccs.length > 0 && {
        ccRecipients: ccs.map((cc) => ({ emailAddress: { address: cc } })),
      }),
      ...(bccs.length > 0 && {
        bccRecipients: bccs.map((bcc) => ({ emailAddress: { address: bcc } })),
      }),
      attachments,
    },
    saveToSentItems: "true",
  };

  const res = await Postman.post(url, emailMessage, this.accessToken);
  console.log(res.data);
}
```

**📌 ポイント**

>* 添付ファイルを `base64` 読み込み添付する

# 2. HTMLテンプレートの実装

```html:自動処理通知.html
<!doctype html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1.0" />
  <title>自動処理完了のお知らせ</title>
  <style>
    body {
      font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
      font-size: 16px;
      color: #333;
      background-color: #f4f4f4;
      margin: 0;
      padding: 0;
    }
    .container {
      max-width: 600px;
      margin: 20px auto;
      background-color: #fff;
      padding: 30px;
      border-radius: 8px;
      box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    }
    h1 {
      font-size: 24px;
      color: #222;
      margin: 0 0 20px 0;
      border-bottom: 1px solid #eee;
      padding-bottom: 10px;
    }
    .important {
      font-weight: bold;
      color: #c0392b;
    }
    .details {
      background-color: #f9f9f9;
      padding: 15px;
      border-radius: 4px;
      margin-bottom: 20px;
    }
    .footer {
      font-size: 12px;
      color: #777;
      text-align: center;
      margin-top: 30px;
      padding-top: 20px;
      border-top: 1px solid #eee;
    }
    @media screen and (max-width: 600px) {
      .container {
        padding: 20px;
      }
    }
  </style>
</head>
<body>
  {{MAIL_CONTEXT}}
</body>
</html>
```

# 3. HTMLテンプレートの差し込み

```ts
const mailHtml = fs.readFileSync("assets/mail/自動処理通知.html", {
  encoding: "utf8",
});

const mailContext = `
<div class="container">
  <h1>【自動処理通知】バリデーションエラーが発生しました</h1>
  <p class="important">※このメールは自動送信です。返信はできません。</p>
  <p>お疲れ様です。</p>
  <p>${errMessage}</p>

  <p><strong>対象ファイル:</strong> ${fileName}</p>

  ${details.map(
    (d) => `
        <div class="details">
            <p><strong>処理名：</strong>${d.処理名}</p>
            <p><strong>処理日時：</strong>${d.処理日時}</p>
            <p><strong>処理ステータス：</strong>${d.処理ステータス}</p>
        </div>
  `
  ).join("")}

  <div class="footer">
    <p>ご不明な点がございましたら、担当部署までご連絡ください。</p>
    <p>
      <a href="mailto:support@example.com">support@example.com</a>
    </p>
    <p>よろしくお願いいたします。</p>
    <p>&copy; ${dayjs().format("YYYY")} Your Company. All rights reserved.</p>
  </div>
</div>
`;

const body = mailHtml.replace("{{MAIL_CONTEXT}}", mailContext);
await Graph.sendMail(
    "自動処理完了のお知らせ",
    ["メールアドレス"],
    ["メールアドレス"],
    ["メールアドレス"],
    body,
    ["添付ファイルのパス"]
);
```

# まとめ

Microsoft Graph APIを活用すれば、Outlookから手動でメールを送らなくても、テンプレートとデータを組み合わせて自動で丁寧なメールを送信する仕組みを作ることができます。

日々の業務報告やツールエラー通知に応用できるので、ぜひ参考にしてみてください。
