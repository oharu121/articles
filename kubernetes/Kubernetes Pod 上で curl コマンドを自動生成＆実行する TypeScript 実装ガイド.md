# はじめに

Kubernetes 上の Pod に対して `curl` を実行する際、毎回手動でコマンドを入力したり、ログを残す処理を書くのは面倒です。この記事では、以下の課題を解決するために作成した TypeScript 実装をご紹介します。

* ✅ 複雑な `curl` コマンドを簡潔に組み立てたい
* ✅ Pod への `kubectl exec` をスクリプト化したい
* ✅ 実行ログを記録・再利用したい

以下の記事で紹介した `Remote` クラスと連携して使用する構成です。
👉 [Remote の実装記事はこちら](https://qiita.com/oharu121/items/770808ec6d5fc0b55f97)

**使用技術スタック**

* TypeScript
* ssh2（リモート実行）
* Kubernetes (`kubectl exec`)
* curl（API 通信）

# コマンド組み立てクラス：`CommandBuilder`

### 1. クラスの構成

`CommandBuilder` は、Pod 上で `kubectl exec` によって `curl` を実行するコマンドを**動的に構築**するためのビルダーパターンのクラスです。

```ts:CommandBuilder.ts
class CommandBuilder {
  private config: string[] = [];
  private command: string[] = [];
}

export default new CommandBuilder();
```

**📌 ポイント**

>- `config`：`kubectl exec` などのベース部分を格納。
>- `command`：`curl` コマンドのパラメータ部分を構築。

- 使用例

```ts
const cmd = CommandBuilder
  .pod("my-pod")
  .patch()
  .token("abc123")
  .content("json")
  .payload({ foo: "bar" })
  .api("http://localhost/api")
  .create();
```

このようにチェーンメソッドで記述できます。

### 2. メソッドチェーンの実装

```ts:CommandBuilder.ts
public reset(): void {
  this.command = [];
}
```

>- `command` をリセット。`config` はそのまま保持する。

```ts:CommandBuilder.ts
public pod(podName: string): void {
  this.reset();
  this.config.push(`kubectl exec`);
  this.config.push(podName);
  this.config.push(`-- curl -Lgkv`);
}
```

>- Pod 名を指定して `kubectl exec` + `curl` のコマンド基盤を構成。
>- `curl` は `-L`（リダイレクト追従）, `-g`（glob無効）, `-k`（SSL検証無効）, `-v`（verbose）を指定。

```ts:CommandBuilder.ts
public patch(): CommandBuilder {
  this.command.push(`-X PATCH`);
  return this;
}
```

>- `PATCH` リクエストを指定するオプション。

```ts:CommandBuilder.ts
public token(token: string): CommandBuilder {
  this.command.push(`-H "Authorization: Bearer ${token}"`);
  return this;
}
```

>- Bearer トークンによる認証ヘッダーを追加。

```ts:CommandBuilder.ts
public header(header: string): CommandBuilder {
  this.command.push(`-H "${header}"`);
  return this;
}
```

>- 任意のヘッダーを追加。

```ts:CommandBuilder.ts
public content(content: string = "json"): CommandBuilder {
  switch(content) {
    case "form":
      this.command.push(`-H 'Content-Type: multipart/form-data'`);
      break;
    case "patch":
      this.command.push(`-H 'Content-Type: application/json-patch+json'`);
      break;
    default:
      this.command.push(`-H 'Content-Type: application/json'`);
  }
  return this;
}
```
>- `Content-Type` ヘッダーを用途に応じて追加。
>  `"form"`：ファイルアップロード用途。
>  `"patch"`：JSON Patch。
>  それ以外：JSON 通信。

```ts:CommandBuilder.ts
public payload(payload: object): CommandBuilder {
  this.command.push(`-d '${JSON.stringify(payload)}'`);
  return this;
}
```

>- リクエストボディを JSON 文字列として追加。

```ts:CommandBuilder.ts
public api(api: string): CommandBuilder {
  this.command.push(`${api}`);
  return this;
}
```

>- 実行する API のエンドポイント URL を追加。

```ts:CommandBuilder.ts
public formMeta(meta: string): CommandBuilder {
  this.command.push(`-F "${meta};type=application/json"`);
  return this;
}
```

>- multipart フォームの一部として JSON メタ情報を送る。

```ts
public formFile(file: string): CommandBuilder {
  this.command.push(`-F "${file};type=application/pdf"`);
  return this;
}
```

>- multipart フォームの一部としてファイル（PDF）を送信。

```ts
public create(): string {
  return [...this.config, ...this.command].join(" ");
}
```
>- `kubectl exec ... curl ...` の全体コマンド文字列を組み立てて返す。


**主なメソッド解説**

| メソッド                                | 役割                                              |
| ----------------------------------- | ----------------------------------------------- |
| `pod(name)`                         | 実行対象の Pod を設定し、`kubectl exec` と `curl` の基本構文を構築 |
| `patch()` / `get()` / `post()` など   | HTTP メソッドの指定                                    |
| `token(token)`                      | Bearer トークン認証ヘッダー追加                             |
| `header(header)`                    | 任意のヘッダーを追加                                      |
| `content(type)`                     | Content-Type を用途別に指定（`json`, `form`, `patch`）   |
| `payload(obj)`                      | JSON ボディをリクエストに含める                              |
| `formMeta(meta)` / `formFile(file)` | multipart/form-data の構築                         |
| `api(url)`                          | API エンドポイントの指定                                  |
| `create()`                          | 最終的なコマンド文字列の出力                                  |

💡 **リセットの仕組み**：
メソッドチェーンによる使い回しで副作用が出ないよう、`reset()` メソッドを用意しています。

# Pod 上でコマンドを実行するユーティリティ：`Utility`


`Utility` クラスは、`CommandBuilder` を使って `curl` コマンドを生成し、それを `Remote.execRemote()` 経由で Pod 上で実行、ログ記録やリトライ処理まで一貫して担います。

**処理フロー**

1. Pod 名取得（キャッシュ）: `checkPodname()`
2. トークン取得: `Postman.getToken()`
3. コマンド生成: `CommandBuilder.xxx()`
4. コマンド実行: `Remote.execRemote()`
5. 結果処理とログ記録: `handleApiResponse()` & `PodLogger`

### 1. `@reset()` デコレータ

```ts:Utility.ts
function reset() {
  return function (_target, _key, descriptor) {
    const original = descriptor.value;
    descriptor.value = function (...args) {
      CommandBuilder.reset();
      return original.apply(this, args);
    };
    return descriptor;
  };
}
```
>- メソッド実行前に `CommandBuilder.reset()` を自動的に挿入するデコレータ。
>- コマンドビルダーの使い回しによる副作用を防止。

### 2. クラスの構成

```ts:Utility.ts
class Utility {
  public podName?: string;
}

export default new Utility();

```
>- Kubernetes の Pod 名を保持。
---

### 3. Pod 名取得

```ts:Utility.ts
private async checkPodname() {
  if (!this.podName) {
    const podName = (await Pod.getPodName())[0] || "";
    this.podName = podName;
    CommandBuilder.pod(podName);
  }
}
```
>- 毎回Pod名にビルド番号が変わるので、最新のものを取得する
>- Pod名が未取得の場合に取得し、CommandBuilder に設定。

### 4. 応答解析 & ログ処理

```ts
private async handleApiResponse(raw: string, callback: (data: any) => void): Promise<void> {
  try {
    const response = JSON.parse(raw);
    await callback(response);
    PodLogger.response(response);
  } catch {
    PodLogger.failed();
    PodLogger.response({ msg: raw });
  }
}
```

>- JSON をパースして結果を渡す。
>- パース失敗時は `failed()` を記録。

### 5. APIメソッドの実装（例）

- 支払いステータスを更新するAPI

```ts:Utility.ts
@reset()
public async updatePaymentStatus(): Promise<void> {
  this.checkPodname();

  const api = apis.UPDATE_PAYMENT_STATUS;
  const token = await Postman.getToken();
  const cmd = CommandBuilder.content("json")
    .header("correlationId:12345")
    .header("requestorUserId:random_user")
    .token(token)
    .api(api)
    .create();

  const result = await Remote.execRemote(cmd);
  const analyze = (data: { numberOfScanned: number }) => {
    if (data.numberOfScanned > 0) PodLogger.success();
  };
  await this.handleApiResponse(result, analyze);

  PodLogger.api(api);
  await PocketBase.writePodLog();
}
```

>- `支払ステータス更新API` を呼び出すためのコマンドを組み立てて実行。
>- 成功判定後に Pod ログを保存。

- KYC書類をアップロードするAPI

```ts:Utility.ts
@reset()
public async uploadFile(fileName: string): Promise<string> {
  this.checkPodname();

  const api = apis.UPLOAD_FILE_TO_DMS;
  const payload = PodLogger.getPayload();
  const jsonString = JSON.stringify(payload).replace(/"/g, '\\"');
  const token = await Postman.getToken();
  const cmd = CommandBuilder.content("form")
    .token(token)
    .header("requestorUserId:123")
    .formMeta(`listOfFile=${jsonString}`)
    .formFile(`fileStream=@/tmp/${fileName}`)
    .api(api)
    .create();

  const result = await Remote.execRemote(cmd);
  let fileId = "";
  const analyze = async (res: FileResponse) => {
    if (res.responseCode === "200") {
      PodLogger.success();
      fileId = res.listOfFiles[0]!.documentIdentifier;
    } else {
      await this.uploadOrderKYC(fileName); // 再帰的リトライ
    }
  };
  await this.handleApiResponse(result, analyze);

  PodLogger.api(api);
  await PocketBase.writePodLog();
  return fileId;
}
```

>- ファイルを DMS にアップロードし、`documentIdentifier` を取得。
>- 成功時に ID を取得、失敗時はリトライ。

# おわりに

本記事では、Kubernetes 上での `curl` 実行を効率化するための `CommandBuilder` ＋ `Utility` 構成を紹介しました。

Pod 上での API テストやデバッグ、あるいは業務バッチの自動化など、多くのユースケースに応用可能です。
ご自身の環境に合わせてカスタマイズしていただければ幸いです。

**今後の展望と拡張アイデア**

* `CommandBuilder` に `GET`, `PUT`, `DELETE` などの追加
* `PodLogger` の標準化と構造整理（例えば統一的なログ形式）
* コマンド生成から実行・ログ出力まで含めた CLI ツール化
* テスト用のモック `Pod` 環境や dry-run 機能の追加