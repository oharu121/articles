# はじめに

Excelファイルを自動で編集・加工したい──そんなとき、JavaScript界隈では [`exceljs`](https://www.npmjs.com/package/exceljs) など便利なライブラリがあります。

ただし、実務で使うには以下のような落とし穴も：

> ❌ ExcelJSで既存テンプレートを加工 → **メタデータが破損して Excel で開けない**

このような課題を解決するため、この記事では **PowerShell × Node.js** を組み合わせた、「ネイティブなExcel操作」の仕組みを紹介します。

>* PowerShellの**COMオブジェクト**で本物のExcelを操作
>* Node.jsで業務ロジックを管理
>* `.ps1` を一時生成して BOM付きで実行 → 日本語や長文でも安心

**🔍 なぜPowerShell + Node.jsなのか？**

>* COMでネイティブ操作：Excelアプリそのものを操作（印刷範囲、保護など）
>* Node.jsで柔軟な処理制御：申込情報の整形や条件分岐など業務ロジックを記述
>* `exceljs`とのハイブリッドも可： 読み込みは`exceljs`、出力はPowerShellで堅牢に
>* 長文スクリプトもOK：**一時ファイル** + **UTF-8 BOM付きで保存**
>→ 実行時エラーを防ぐ

# PowerShellラッパークラスの実装

### 1. 実装概要

まずはPowerShell実行のラッパーを作成します。

```ts:Powershell.ts
interface PowerShellResult {
  stdout: string;
  stderr: string;
  exitCode: number;
}

class Powershell {
  // メソッドは後述
}

export default new Powershell();
```

📌 ポイント：

>* `PowerShellResult` 型を先に定義しておくと、各処理の共通化やエラーハンドリングが楽になります。

### 2. PowerShell 実行

```ts:Powershell.ts
class PowerShell {
  public async executePowerShell(
    script: string,
    args: string[] = []
  ): Promise<PowerShellResult> {
    const tempScriptPath = `${os.tmpdir()}/temp-file.ps1`;
    const bomScript = "\uFEFF" + script;
    fs.writeFileSync(tempScriptPath, bomScript);

    return new Promise((resolve, reject) => {
      const ps = spawn("powershell.exe", [
        "-ExecutionPolicy", "Bypass",
        "-File", tempScriptPath,
        ...args,
      ]);

      let stdout = "";
      let stderr = "";

      ps.stdout.on("data", (data: Buffer) => (stdout += data.toString()));
      ps.stderr.on("data", (data: Buffer) => (stderr += data.toString()));

      ps.on("close", (code: number) => {
        fs.unlinkSync(tempScriptPath); // 後始末

        const result = { stdout, stderr, exitCode: code };
        code !== 0 ? reject(result) : resolve(result);
      });

      ps.on("error", (err: Error) => reject(err));
    });
  }
}

```

📌 ポイント：

>* `ExecutionPolicy: Bypass` により、制限のある環境でもスクリプトを実行可能に
>* `stdout`, `stderr` を明示的に取得して、ログやエラー解析に利用
>* `\uFEFF` でUTF-8 BOMを付けないと日本語が文字化けする！
>* コマンドラインから渡すと、長すぎてPowerShellがコケる
>→ **スクリプトをファイルに書き出して `-File` で実行**するのがベスト

# 活用例
### 1. 業務クラスの構成：`CreateContract.ts`

申込データを取得し、加工対象のパーツと値のマッピングを行った上で PowerShell を呼び出します。

```ts:CreateContract.ts
class CreateContract {
    public async main() {
        // 申込データを取得する
        await this.getApplicationData();
        // パーツシートのどの範囲（開始行～終了行）をコピーするかを定義
        await this.readPartsMap();
        // パーツシートのどのセルに値を入れるかを定義
        await this.readPrintMap();
        // PowerShellスクリプトを実行する
        await this.executePowerShell();
    }
}
```

### 2. PowerShellスクリプトの生成と実行
```CreateContract.ts
public async executePowerShell() {
    const partsCommand: string[] = [];

    this.add配送情報(this.applicationData, partsCommand);
    this.addカード情報(this.applicationData, partsCommand);
    this.add別紙(, partsCommand);

    const script = this.buildScript("[出力パス]", partsCommand);
    
    const result = await this.executePowerShell(script);
    if (result.exitCode !== 0) {
      Logger.error("PowerShell script failed");
      console.log(result.stderr);
      console.log(result.stdout);
    }
}
```

### 3. PowerShellスクリプトの構築

`partsCommand` を挿入してテンプレートを動的に操作するスクリプトを生成します。

```ts:CreateContract.ts
public buildScript(
    destPath: string,
    partsCommand: string[]
  ) {
    const resMain = path.resolve("[テンプレートのパス]");
    const resPart = path.resolve("[パーツのパス]");
    const resDest = path.resolve(destPath);

    const script = `
        $OutputEncoding = [System.Text.Encoding]::UTF8
        $InputEncoding = [System.Text.Encoding]::UTF8
        
        try {
          $excel = New-Object -ComObject Excel.Application
          $excel.Visible = $false
        
          $mainWorkbook = $excel.Workbooks.Open("${resMain}", $null, $true)
          $mainSheet = $mainWorkbook.Worksheets.Item("[シート名]")
        
          $partsWorkbook = $excel.Workbooks.Open("${resPart}", $null, $true)
          $partsSheet = $partsWorkbook.Worksheets.Item("[シート名]")
        
          ${partsCommand.join("\n")}
          [System.Runtime.InteropServices.Marshal]::ReleaseComObject($partsSheet) | Out-Null
        
          $mainSheet.PageSetup.PrintArea = "A1:AP$rowNum"
          [System.Runtime.InteropServices.Marshal]::ReleaseComObject($mainSheet) | Out-Null
        
          $mainWorkbook.SaveCopyAs("${resDest}")
          $mainWorkbook.Close($false)
          [System.Runtime.InteropServices.Marshal]::ReleaseComObject($mainWorkbook) | Out-Null
        
          $partsWorkbook.Close($false)
          [System.Runtime.InteropServices.Marshal]::ReleaseComObject($partsWorkbook) | Out-Null
        
          $excel.Quit()
          [System.Runtime.InteropServices.Marshal]::ReleaseComObject($excel) | Out-Null
        }
        catch {
          Write-Error "Error: $($_.Exception.Message)"
          exit 1
        }
        finally {
          if ($mainSheet) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($mainSheet) | Out-Null }
          if ($mainWorkbook) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($mainWorkbook) | Out-Null }
          if ($partsSheet) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($partsSheet) | Out-Null }
          if ($partsWorkbook) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($partsWorkbook) | Out-Null }
          if ($insertRange) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($insertRange) | Out-Null }
          if ($partsRange) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($partsRange) | Out-Null }
          if ($excel) { [System.Runtime.InteropServices.Marshal]::ReleaseComObject($excel) | Out-Null }
        
          [System.GC]::Collect()
          [System.GC]::WaitForPendingFinalizers()
        }
    `;

    return script;
  }
```
📌 ポイント：
>* `$excel.Visible = $false` でヘッドレス
>* `Open(..., $null, $true)` は読み取り専用
>* `Close($false)` で保存せず元のファイルを閉じる
>* `ReleaseComObject` メモリリーク防止

### 4. パーツの動的挿入処理例

例：カード情報を条件付きで挿入

```ts:CreateContract.ts
private addカード情報(data: ApplicationData, partsCommand: string[]) {
    if (!data.購入方法.カード払い) return;

    const { start, end } = this.partsMap.get("カード情報")!;
    
    const カード番号 = {
      cell: this.printMap.get("カード番号"),
      value: data.購入方法.カード番号,
    };

    const 有効期限 = {
      cell: this.printMap.get("有効期限"),
      value: data.購入方法.有効期限,
    };

    const setValues = [カード番号, 有効期限]
      .filter((item) => item.value || item.value === 0)
      .map((item) =>
        typeof item.value === "string"
          ? `$partsSheet.Range("${item.cell}").Value2 = "${item.value}"`
          : `$partsSheet.Range("${item.cell}").Value2 = ${item.value}`
      )
      .join("\n");

    const command = `
        $rowNum += 1
        ${setValues}
        
        $partsRange = $partsSheet.Range("A${start}:AQ${end}")
        $mainSheet.Rows("\${rowNum}:$($rowNum + (${end} - ${start}))").Insert()
        
        $insertRange = $mainSheet.Range("A$rowNum")
        $partsRange.Copy($insertRange)
        
        $rowNum += (${end} - ${start})
    `;
    
    partsCommand.push(command);
  }
}
```
📌 ポイント：
>* `.filter((item) => item.value || item.value === 0)` でパーツの編集箇所を決める
>* パーツを編集してからテンプレートへ挿入する
>* `typeof item.value === "string"` で文字列を `""` で囲む

# おわりに

PowerShellを活用すれば、Excelを"本物のExcelアプリ"として自動操作できます。JavaScriptだけでは対応できなかった要件も、Node.jsとの組み合わせで**コード管理のしやすさと安定性**の両立が可能に。

帳票生成や申込書作成、パーツの差し替え処理など、**実務の現場で活きる強力な自動化手段**として、ぜひ取り入れてみてください。
