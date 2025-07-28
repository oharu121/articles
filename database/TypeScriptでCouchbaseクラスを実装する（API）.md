# TypeScriptでCouchbaseクラスを実装する（API）

# はじめに

社内ツールなどでCouchbaseを操作する必要があるとき、シンプルにAPI経由で読み書きできるクラスを用意しておくと便利です。この記事では、CouchbaseのREST APIおよびN1QLクエリを `curl` で叩くことでデータを操作する `Couchbase` クラスの実装を紹介します。

* **対象読者**: Node.js/TypeScriptで社内ツールやスクリプトを作っている方
* **ゴール**: Couchbaseに対して「ドキュメントの取得」「クエリの実行」「更新処理」などができる汎用クラスの作成

**❓ なぜSDKを使わないのか**

Couchbaseには便利なSDKがありますが、社内ではSSHを介してデータベースを操作する必要があるので、本記事では APIを直接叩く構成を採用しています。

https://qiita.com/oharu121/items/770808ec6d5fc0b55f97

# 1. 認証と環境ごとの設定を整理する

環境変数に応じてホストと認証情報を切り替えています。

```ts:class/Couchbase.ts
class Couchbase {
  private host(): string {
    const env = SessionDB.getEnv();
    switch (env) {
      case "prod":
        return hosts.COUCHBASE_HOST_PROD;
      case "stg":
        return hosts.COUCHBASE_HOST_STG;
      default:
        throw new Error("Unknown environment");
    }
  }

  private auth(): string {
    const env = SessionDB.getEnv();
    return env === "prod"
      ? `${credentials.COUCHBASE_USERNAME_PROD}:${credentials.COUCHBASE_PASSWORD_PROD}`
      : `${credentials.COUCHBASE_USERNAME_STG}:${credentials.COUCHBASE_PASSWORD_STG}`;
  }
}

export default new Couchbase(); // singleton
```

# 2. ドキュメントの取得

### REST APIを使う場合

```ts:class/Couchbase.ts
class Couchbase {
  // 前略...
  public async getJsonByDocId(
    id: string,
    bucket: string
  ): Promise<ResponseCouchbase> {
    const result = await Remote.execJump(
      `curl -X GET -u ${this.auth()} ${this.host()}:8091/pools/default/buckets/${bucket}/docs/${id}`
    );

    if (!result) {
      return {
        complete: false,
        result: {},
        msg: "The response from Couchbase is empty",
      };
    }

    const resultJson = JSON.parse(result);

    if (resultJson.status === "success") {
      const jsonString = CommonUtil.prettyJson(result);
      return {
        complete: true,
        result: JSON.parse(jsonString).json,
        msg: "",
      };
    } else {
      return {
        complete: false,
        result: {},
        msg: resultJson.errors?.[0]?.msg ?? "Unknown error",
      };
    }
  }
}

export default new Couchbase(); // singleton
```

取得結果を `prettyJson` で整形し、 `status` を見て成否判定しています。

### N1QLで取得する場合

```ts:class/Couchbase.ts
class Couchbase {
  // 前略...
  public async getDocumentById(id: string, bucket: string): Promise<object> {
    const query = `SELECT * FROM \`${bucket}\` USE KEYS "${id}"`;
    const result = await this.execReadQuery(query);

    if (result.complete) {
      const searchResult = result.result as { [key: string]: object }[];
      if (!searchResult[0]) {
        throw new Error(`Document with id: ${id} not found.`);
      }

      const document = searchResult[0][bucket];
      if (document) {
        return document;
      } else {
        throw new Error(
          "Unexpected return type for document, expecting object."
        );
      }
    } else {
      Logger.error(`Failed to retrieve ${id}`);
      throw new Error(result.msg);
    }
  }
}

export default new Couchbase(); // singleton
```

N1QLを使うと条件付きの取得やJOINが可能になります。

# 3. N1QLクエリの実行

### 読み取り

```ts:class/Couchbase.ts
class Couchbase {
  // 前略...
  private async execReadQuery(n1qlQuery: string): Promise<ResponseCouchbase> {
    const cmd = [
      `curl -g`,
      `-u ${this.auth()}`,
      `${this.host()}:8093/query/service`,
      `--data-urlencode 'statement=${n1qlQuery}'`,
    ].join(" ");

    const result = await Remote.execJump(cmd);

    if (!result) {
      return {
        complete: false,
        result: {},
        msg: "The response from Couchbase is empty",
      };
    }

    const resultJson = JSON.parse(result);

    if (resultJson.status === "success") {
      return {
        complete: true,
        result: resultJson.results,
        msg: "",
      };
    } else {
      return {
        complete: false,
        result: {},
        msg: resultJson.errors?.[0]?.msg ?? "Unknown error",
      };
    }
  }
}

export default new Couchbase(); // singleton
```

### 書き込み

```ts:class/Couchbase.ts
class Couchbase {
  // 前略...
  public async execWriteQuery(n1qlQuery: string): Promise<ResponseCouchbase> {
    const cmd = [
      `curl -g`,
      `-u ${this.auth()}`,
      `${this.host()}:8093/query/service`,
      `--data-urlencode 'statement=${n1qlQuery}'`,
    ].join(" ");

    const result: string = await Remote.execJump(cmd);

    if (!result) {
      return {
        complete: false,
        result: {},
        msg: "The response from Couchbase is empty",
      };
    }

    const resultJson = JSON.parse(result);
    if (resultJson.status === "success") {
      return {
        complete: true,
        result: {},
        msg: "",
      };
    } else {
      return {
        complete: false,
        result: {},
        msg: resultJson.errors?.[0]?.msg ?? "Unknown error",
      };
    }
  }
}

export default new Couchbase(); // singleton
```

成功時は `status === "success"` をチェックして結果を返します。

# 4. よく使うユースケース例

```ts:class/Couchbase.ts
class Couchbase {
  // 前略...
  // 取得する
  public async getOrderDocVersion(orderId: string): Promise<string> {
    const query = [
      `SELECT TO_STRING(META().cas) AS cas`,
      `FROM \`${names.DATA_BUCKET}\``,
      `WHERE type = "order"`,
      `AND order_id = "${orderId}"`,
    ].join(" ");

    const result = await this.execReadQuery(query);
    if (result.complete) {
      const searchResult = result.result as { cas: string }[];
      if (searchResult[0]?.cas) {
        return searchResult[0].cas;
      } else {
        throw new Error("Unexpected return type for cas.");
      }
    } else {
      Logger.error(`Failed to retrieve order version for ${orderId}`);
      throw new Error(result.msg);
    }
  }

  // 読み取りクエリ
  public async getCustomerId(orderId: string): Promise<string> {
    const query = [
      `SELECT id`,
      `FROM \`${names.ORDER_BUCKET}\``,
      `WHERE type = "customerOrder"`,
      `AND external_order_id = "${orderId}"`,
    ].join(" ");

    const result = await this.execReadQuery(query);

    if (result.complete) {
      const searchResult = result.result as { id: string }[];
      if (searchResult[0]?.id) {
        return searchResult[0].id;
      } else {
        throw new Error("Unexpected return type for customerId.");
      }
    } else {
      Logger.error(`Failed to retrieve customerId for ${orderId}`);
      throw new Error(result.msg);
    }
  }

  // 書き込みクエリ
  public async changeMethod(
    accountId: string,
    timeStamp: number
  ): Promise<void> {
    const key = `payment::${accountId}`;
    const query = [
      `UPDATE \`${names.BUSINESS_BUCKET}\``,
      `USE KEYS "${key}"`,
      `SET scheduler_ts = NULL,`,
      `    switch_date = ${timeStamp}`,
    ].join(" ");

    await this.execWriteQuery(query);
  }
}

export default new Couchbase(); // singleton
```

Couchbaseではdocument IDにprefixをつけて管理するのが一般的なので、`ORDER::`のような形式で操作しています。

# おわりに

プロジェクトにおいてCouchbaseとのインターフェースが必要な場面で、今回紹介した `Couchbase` クラスは再利用可能な基盤となります。curlベースの実装なので依存ライブラリもなく、他のDBアクセス方法にも応用が利きます。

「N1QLクエリをどこまで任せるか」「データ構造をどう抽象化するか」など、より堅牢な設計も今後検討していけると良いでしょう。
