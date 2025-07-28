# PocketBase × JSフック 拡張術

# はじめに
[**PocketBase**](https://pocketbase.io/) は、Go製の軽量バックエンド。単体バイナリで API + 管理画面 + DB（SQLite）が起動でき、**フロントエンド開発者にとって理想的な開発環境**を提供してくれます。

本記事では、**PocketBase公式SDKを使わずに、REST APIを直接叩く構成**での実装方法を紹介します。TypeScript + axios を用いて、起動チェックからAPI呼び出し、JSフックによる拡張までを一気通貫で解説します。

**❓ なぜSDKを使わないのか**

PocketBaseには便利なSDKがありますが、TypeScriptプロジェクトで使用した際に型エラーや動作不良が頻発しました。そこで、本記事では **APIを直接叩く構成**を採用しています。

# PocketBaseのセットアップ
**📦 PocketBaseをダウンロードする**

[公式サイト](https://pocketbase.io/docs/)からOSに応じたPocketBaseのバイナリをダウンロードします。
プロジェクト内に `pocketbase/` フォルダを作成し、バイナリ（`pocketbase.exe`や`pocketbase`）を格納します。

https://pocketbase.io/docs/

**🌐 管理画面にアクセス**

以下のコマンドでサーバーを起動します。

```
pocketbase/pocketbase serve
```

> ⚠️ セキュリティソフトにブロックされた場合は、許可してください。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/f31744b5-1301-168e-a5f9-5e9a5da7cefb.png)

立ち上がったら`http://127.0.0.1:8090/_/`にアクセスします。
アカウント作成を行います。

> ⚠️認証メール来ないので、Emailは適当でいいで実在する必要はありません。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/f30ec4fc-b837-e644-9550-187ce582e767.png)

「Create and login」ボタンを押して、ログインします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/874dedbc-ff30-8f56-e093-77eba7ccf9a9.png)

必要に応じてテーブルを作成します。（公式より）
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/95c3a1b8-1393-4434-befc-f0b35261e79d.png)

# 1. PocketBase の起動処理（クラス外）
ここで実装する内容はクラスの外です。

**🌐 起動状態の確認**

PocketBase がすでに起動済みかどうかをチェックする関数です。

```ts:class/Pocketbase.ts
export async function checkRunning(): Promise<boolean> {
  return new Promise((resolve) => {
    const options = {
      hostname: "localhost",
      port: 8090,
      path: "/_/",
      method: "GET",
    };

    const req = http.request(options, (res) => {
      resolve(res.statusCode === 200);
    });

    req.on("error", () => {
      resolve(false);
    });

    req.end();
  });
}
```

**🔁 起動済みでなければ初期化**

```ts:class/Pocketbase.ts
async function checkConnection(): Promise<void> {
  const isRunning = await checkRunning();
  if (isRunning) return;

  await initPocketBase();
}
```

**🔧 PocketBase の起動処理**

まずは OS 判定をして、適切なパスから PocketBase を起動します。

```ts:class/Pocketbase.ts

export function initPocketBase(): Promise<void> {
  const pocketbasePath =
    environment.CURRENT_OS === "win32"
      ? paths.POCKETBASE_PATH_WIN
      : paths.POCKETBASE_PATH_MAC;

  return new Promise<void>((resolve, reject) => {
    let pocketBaseProcess;

    try {
      // macOS: 実行権限付与とquarantine解除
      if (environment.CURRENT_OS === "darwin") {
        execSync(`chmod +x "${paths.POCKETBASE_PATH_MAC}"`);
        execSync(
          `xattr -d com.apple.quarantine "${paths.POCKETBASE_PATH_MAC}"`
        );
      }

      pocketBaseProcess = spawn(pocketbasePath, ["serve"]);
    } catch (error) {
      Logger.error(`exec error: ${error}`);
      return reject(error);
    }

    if (!pocketBaseProcess) return;

    pocketBaseProcess.stdout.on("data", (chunk) => {
      const message = chunk.toString();

      if (message.includes("Server started at http://127.0.0.1:8090/")) {
        Logger.success("PocketBase is running!");
        resolve();
      }
    });

    pocketBaseProcess.stderr.on("data", (chunk) => {
      Logger.error(chunk.toString());
    });

    pocketBaseProcess.on("close", (code) => {
      if (code !== 0) {
        reject(new Error(`PocketBase process exited with code ${code}`));
      }
    });
  });
}
```

**🎁 デコレータで自動実行を仕込む**

関数実行前に PocketBase が起動していることを保証するデコレータを作成。

```ts:class/Pocketbase.ts
function pocketbase() {
  return function (
    _target: object,
    _key: string | symbol,
    descriptor: PropertyDescriptor
  ) {
    const original = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      await checkConnection();
      return await original.apply(this, args);
    };

    return descriptor;
  };
}
```

# 2. 標準機能で実装する
以下は、「クラスによるサービス層の設計と活用法」を中心に据えた内容になります。

今回は、**PocketBase API を効率的に操作するためのクラス構成**を紹介します。
**認証・一覧取得・重複チェック・ログ削除など、API 呼び出しを抽象化**し、再利用性の高い設計にしています。

* `@pocketbase()` デコレータ：メソッド呼び出し前に PocketBase の起動確認を実行
* `getAuthToken()`：管理者認証を内部で取得し、Bearer トークンを返す
* 各メソッドは PocketBase の API に対応し、axios 経由でアクセス

**🧬 認証**

```ts:class/Pocketbase.ts
class PocketBase {
  private auth = {
    identity: credentials.POCKETBASE_USERNAME,
    password: credentials.POCKETBASE_PASSWORD,
  };

  @pocketbase()
  protected async getAuthToken() {
    const url = `${apis.POCKETBASE_BASE_URL}/api/admins/auth-with-password`;
    const authResponse = await axios.post(url, this.auth);
    return authResponse.data.token;
  }
}

export default new Pocketbase(); // singleton
```

📌 よく使う機能まとめ

```ts:class/Pocketbase.ts
class PocketBase {
  // ...前略
  // コレクションを表示する
  @pocketbase()
  public async listCollections() {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections`;
    const token = await this.getAuthToken();
    const headers = { headers: { Authorization: `Bearer ${token}` } };

    const response = await axios.get(url, headers).catch((err) => {
      throw new Error(err);
    });

    console.log(response.data);
  }

  // 重複チェック
  @pocketbase()
  private async checkForDuplicate(id: string): Promise<boolean> {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections/foo/records?filter=id='${id}'`;
    const token = await this.getAuthToken();
    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    const response = await axios.get(url, headers).catch((err) => {
      console.log(err);
      throw new Error(err);
    });

    return response.data.items.length > 0;
  }

  // 登録処理（重複スキップ付き）
  @pocketbase()
  public async registerEntries(entries: FooProgress[]) {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections/foo/records`;
    const token = await this.getAuthToken();
    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    for (const entry of entries) {
      const isDuplicate = await this.checkForDuplicate(entry.title);
      if (isDuplicate) {
        Logger.error(`Duplicate found for ${entry.title}`);
        continue;
      }

      await axios.post(url, entry, headers).catch((err) => {
        console.log(err);
        throw new Error(err);
      });

      Logger.success(`Registered ${entry.title} to foo waitlist`);
    }
  }

  // ログ一覧
  @pocketbase()
  public async getAllApiLogs(): Promise<FooProgress[]> {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections/fooLogs/records`;
    const token = await this.getAuthToken();
    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    const response = await axios.get(url, headers);
    return response.data.items;
  }

  // 削除・アーカイブなど
  @pocketbase()
  public async deleteApiLogs(entries: object[]): Promise<void> {
    const token = await this.getAuthToken();
    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    const records = entries as [{ id: string }];
    for (const record of records) {
      const url = `${apis.POCKETBASE_BASE_URL}/api/collections/fooLogs/records/${record.id}`;
      await axios.delete(url, headers);
    }
  }
}

export default new PocketBase(); // singleton
```

# 3. JS拡張機能で実装する
PocketBase は標準で REST API を備えていますが、**もっと柔軟なロジック**（複雑なSQLや独自ルート処理）が欲しい場面もありますよね？

[**PocketBaseのJS拡張機能**](https://pocketbase.io/docs/js-overview/)はPocketbaseのカスタマイズのルーターを定義することでより柔軟にクエリすることが可能です。PocketBase に `*.pb.js` ファイルを追加し、**Node.js で実行できる拡張APIを追加する方法**を紹介します。

**🧠 何ができるの？**

* 任意のルートを追加（`routerAdd("GET", "/api/xxx", handler)`）
* SQLを直接発行（`$app.dao().db().newQuery(...)`）
* POSTパラメータを処理
* データ構造を柔軟に扱う（`newDynamicModel()`）

### セットアップ

**📁 ディレクトリ構成**

追加前：

```
.
├── pocketbase          ← バイナリ (macOS)
├── pocketbase.exe      ← バイナリ (Windows)
├── pb_data             ← DBなどが入る
└── pb_migrations       ← マイグレーション用フォルダ
```

追加後：

```
.
├── pocketbase          ← バイナリ (macOS)
├── pocketbase.exe      ← バイナリ (Windows)
├── pb_data             ← DBなどが入る
├── pb_migrations       ← マイグレーション用フォルダ
└── pb_hooks            ← 🔥 JSフックファイルを入れるフォルダ（今回追加）
    └── custom.pb.js    ← ここに独自 API を記述
```

> `pb_hooks` フォルダは PocketBase 起動時に自動読み込みされ、JSファイル内の `routerAdd()` などのAPI拡張が実行されます。

**✅ 拡張 API の使い方**

PocketBase 起動時、`pb_hooks/*.pb.js` に記述されたコードは自動で読み込まれます。以下のような構文で、任意のルートを追加できます。

**📘 拡張 API の実装例（抽象化済み）**

```js:custom.pb.js
// 🔍 条件付きフィルタ取得
routerAdd("POST", "api/collections/Items/filterByStatus", (c) => {
  const { status, type, limit, afterDays, now } = $apis.requestInfo(c).data;

  const records = arrayOf(newDynamicModel({
    itemId: "",
    label: "",
    status: "",
    createdAt: "",
    updatedAt: "",
  }));

  const query = [
    `SELECT *`,
    `FROM Items`,
    `WHERE status = '${status}'`,
    `AND type = '${type}'`,
    `AND retryCount < 3`,
    afterDays === 0 ? "" : `AND '${now}' >= availableFrom`,
    afterDays === 0 ? "" : `AND '${now}' > lastChecked`,
    afterDays === 0 ? "" : `AND '${now}' >= datetime(registeredAt, '+${afterDays} days')`,
    `ORDER BY createdAt ASC`,
    `LIMIT ${limit}`,
  ].join(" ");

  $app.dao().db().newQuery(query).all(records);

  return c.json(200, {
    message: "Query executed successfully.",
    items: records,
  });
});

// 📋 全件取得
routerAdd("GET", "api/collections/Items/getAll", (c) => {
  const records = arrayOf(newDynamicModel({
    itemId: "",
    label: "",
    status: "",
    createdAt: "",
  }));

  const query = `SELECT * FROM Items`;

  $app.dao().db().newQuery(query).all(records);

  return c.json(200, {
    message: "All items retrieved successfully.",
    items: records,
  });
});

// ❌ データ削除
routerAdd("POST", "api/collections/Items/deleteById", (c) => {
  const itemId = $apis.requestInfo(c).data.itemId;
  const query = `DELETE FROM Items WHERE itemId = '${itemId}'`;

  $app.dao().db().newQuery(query).execute();

  return c.json(200, {
    message: "Item deleted successfully.",
  });
});

// 🔄 ステータス更新
routerAdd("POST", "api/collections/Items/updateStatus", (c) => {
  const itemId = $apis.requestInfo(c).data.itemId;
  const newStatus = $apis.requestInfo(c).data.status;

  const query = [
    `UPDATE Items`,
    `SET status = '${newStatus}'`,
    `WHERE itemId = '${itemId}'`,
  ].join(" ");

  $app.dao().db().newQuery(query).execute();

  return c.json(200, {
    message: "Status updated successfully.",
  });
});
```

### 使い方
**🏁 Pocketbaseクラスに実装する**

1. `pb_hooks/custom.pb.js` に上記コードを保存
1. クラス`Pocketbase.ts`に実装する
2. PocketBase を再起動（`./pocketbase serve`）

```ts:class/Pocketbase.ts
class PocketBase {
  // 前略...
  // 🔍 条件付きフィルタ取得
  @pocketbase()
  public async filterItemsByStatus(params: {
    status: string;
    type: string;
    limit: number;
    afterDays: number;
    now: string;
  }): Promise<FooItem[]> {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections/Items/filterByStatus`;
    const token = await this.getAuthToken();

    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    const response = await axios.post(url, params, headers);
    return response.data.items;
  }

  // 📋 全件取得
  @pocketbase()
  public async getAllItems(): Promise<FooItem[]> {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections/Items/getAll`;
    const token = await this.getAuthToken();

    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    const response = await axios.get(url, headers);
    return response.data.items;
  }

  // ❌ レコード削除
  @pocketbase()
  public async deleteItemById(itemId: string): Promise<void> {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections/Items/deleteById`;
    const token = await this.getAuthToken();

    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    await axios.post(url, { itemId }, headers);
  }

  // 🔄 ステータス更新
  @pocketbase()
  public async updateItemStatus(itemId: string, status: string): Promise<void> {
    const url = `${apis.POCKETBASE_BASE_URL}/api/collections/Items/updateStatus`;
    const token = await this.getAuthToken();

    const headers = {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
    };

    await axios.post(url, { itemId, status }, headers);
  }
}

export default new PocketBase(); // singleton
```

📝 注意点

* `routerAdd()` は GET/POST 両方に対応
* `$app.dao().db().newQuery()` による SQL 実行は **SQLインジェクションに注意**
* 型はゆるいので、整合性やバリデーションは自前で行う必要あり
* ルート名・クエリ構造はなるべく抽象化 or 簡素化しておく
* 本番環境では .pb.js の配置や公開範囲に制限を設けることを推奨

📚 おすすめの活用シーン

* 複雑なフィルター・JOINが必要な集計処理
* 特定の業務ロジックをAPIにカプセル化
* 外部APIとの中継処理を PocketBase 側で行いたいとき
* 標準 GUI では扱いにくいロジックも、サーバーサイドに移譲可能
* SQL を直接実行できるため、パフォーマンス最適化も柔軟に対応

# おわりに

PocketBaseは、軽量ながらもAPIや管理画面、SQLiteベースのDBを一括で提供してくれる非常に便利なバックエンドツールです。
今回紹介したように、公式SDKを使わずともTypeScriptからAPIを直接呼び出すことで、**柔軟かつシンプルな構成**で扱うことができます。

また、`.pb.js` を用いた拡張機能を活用することで、**業務に合わせた独自のAPI設計や複雑なクエリ処理**も可能になり、PocketBaseの可能性はさらに広がります。

今後もプロジェクトに応じて、PocketBaseを軽量なBFF（Backend For Frontend）として活用していければと思います。
