# Node.js × PowerShell で Outlook を操作する

# はじめに

Outlook でメールを自動送信したり、指定のメールを検索したい。

JavaScript の世界では直接 Outlook を操作するライブラリは限られており、実務では Windows 環境の Outlook アプリを直接操作する方法が正確で確実です。

そこで、PowerShell から COM オブジェクトを使って Outlook を操作するスクリプトを Node.js から呼び出す方法を紹介します。

Powershellラッパークラス実装の基本編は下記をご参照ください：

https://qiita.com/oharu121/items/6095699c0a822f483bf9

# Outlook のバージョンを取得する

```ts:Powershell.ts
public async getVersion(): Promise<string> {
  const script = `
  try {
    $outlook = [System.Runtime.InteropServices.Marshal]::GetActiveObject("Outlook.Application")
  } catch {
    $outlook = New-Object -ComObject Outlook.Application
  }

  try {
    $version = $outlook.Version
    Write-Output $version
  } catch {
    Write-Error "Error: $($_.Exception.Message)"
    exit 1
  }
  `;

  const result = await this.executePowerShell(script);
  return result.stdout.trim();
}
```

* 実行環境に Outlook が存在すればバージョン情報を取得
* `Write-Output` の結果は stdout として取得

# Outlook でメールを送信する

```ts:Powershell.ts
public async sendEmail(
  subject: string,
  body: string,
  to: string,
  cc?: string
): Promise<void> {
  let script = `
  try {
    $outlook = [System.Runtime.InteropServices.Marshal]::GetActiveObject("Outlook.Application")
  } catch {
    $outlook = New-Object -ComObject Outlook.Application
  }

  try {
    $mail = $outlook.CreateItem(0)
    $recipient = $mail.Recipients.Add("${to}")
    $recipient.Resolve() | Out-Null

    if ($recipient.Resolved -eq $false) {
      throw "Recipient '${to}' could not be resolved."
    }

    $mail.Subject = "${subject}"
    $mail.Body = "${body}"
    $mail.To = $recipient.Name
  `;

  if (cc) {
    script += `
    $ccRecipient = $mail.Recipients.Add("${cc}")
    $ccRecipient.Resolve() | Out-Null

    if ($ccRecipient.Resolved -eq $false) {
      throw "CC Recipient '${cc}' could not be resolved."
    }
    $mail.CC = $ccRecipient.Name
    `;
  }

  script += `
    $mail.Send()
  } catch {
    Write-Error "Error: $($_.Exception.Message)"
    exit 1
  }
  `;

  const result = await this.executePowerShell(script);
  if (result.exitCode !== 0) {
    throw new Error(`Failed to send email: ${result.stderr}`);
  }
}
```

* `Recipients.Add()` と `Resolve()` の扱いに注意
* CC は実行時に有効なら追加

# 指定メールを検索する

```ts:Powershell.ts
public async searchForEmail(
  targetSubject: string,
  searchTimeWindowMinutes: number
): Promise<object | null> {
  const script = `
  try {
    $outlook = [System.Runtime.InteropServices.Marshal]::GetActiveObject("Outlook.Application")
  } catch {
    $outlook = New-Object -ComObject Outlook.Application
  }

  try {
    $namespace = $outlook.GetNamespace("MAPI")
    $inbox = $namespace.GetDefaultFolder(6)
    $startTime = (Get-Date).AddMinutes(-${searchTimeWindowMinutes})

    $emails = $inbox.Items | Where-Object {
      $_.Class -eq 43 -and
      $_.Subject -eq "${targetSubject}" -and
      $_.ReceivedTime -gt $startTime
    }

    if ($emails) {
      foreach ($email in $emails) {
        Write-Output "Found email with subject '${targetSubject}'"
        Write-Output $email.SenderName
        Write-Output $email.SenderEmailAddress
        Write-Output $email.Body
      }
    } else {
      Write-Output "No email found with subject '${targetSubject}' within last ${searchTimeWindowMinutes} minutes."
    }
  } catch {
    Write-Error "Error: $($_.Exception.Message)"
    exit 1
  }
  `;

  const result = await this.executePowerShell(script);
  const lines = result.stdout.trim().split("\n");

  if (lines[0]?.includes("Found email")) {
    return {
      senderName: lines[1],
      senderEmailAddress: lines[2],
      body: lines[3],
    };
  }

  return null;
}
```

* Subject でフィルター
* `ReceivedTime` を使って時間範囲で検索

# まとめ

* Node.js から Outlook を操作するには PowerShell の COM 操作が最も確実
* 送信や検索は UI なしでバックグラウンドで完結できる
* Write-Output は stdout に、Write-Error は stderr に出力される点に注意

Excel 編と同様に、実務でよく使う Outlook の自動操作をスクリプト化しておくと、操作ミスやヒューマーエラーを防げます。
