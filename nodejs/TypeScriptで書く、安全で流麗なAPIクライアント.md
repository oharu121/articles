# はじめに

API通信は、現代のWebアプリケーション開発に欠かせない要素です。多くの開発者が `axios` のような優れたHTTPクライアントライブラリを利用していますが、プロジェクトが複雑になるにつれて、リクエストごとの設定管理が煩雑になることはありませんか？

「このエンドポイントにはこの認証トークンを、あちらには別のヘッダーを…」

こうした設定をオブジェクトとして管理していると、似たような設定が散乱し、やがてはメンテナンス性の低いコードを生み出す原因にもなります。

そこで本日紹介するのが、こうした課題を解決するために設計された **`Axon`** です。`Axon` は`axios` をラップし、メソッドチェーンによる流れるようなインターフェースと、イミュータブル（不変）な設計によって、**安全で、読みやすく、再利用性の高い** APIリクエストを実現します。

**Axonが提供する3つのコアな体験**

`Axon`の設計思想は、3つのシンプルなコンセプトに基づいています。

>1.  **✨ 流麗なインターフェース (Fluent Interface)**: メソッドを繋げていくことで、どんなリクエストなのかが一目瞭然になります。
>2.  **🛡️ イミュータブルな設計 (Immutable Design)**: 一度作成したクライアントの設定は変更されません。これにより、予期せぬ副作用を防ぎ、安全にクライアントを再利用できます。
>3.  **⛑️ 安全なデフォルト値 (Safe by Default)**: 危険な設定は、開発者が明確に意図した場合にのみ有効化できます。

# Axonの基本的な使い方

百聞は一見にしかず。まずは`Axon`がいかに直感的に使えるかを見てみましょう。

### 1. 基本的なGETリクエスト

`Axon.new()`から始め、最後にHTTPメソッド（`.get()`など）を呼び出すだけです。

```typescript
import Axon from './Axon'; // 作成したAxonクラスをインポート

async function fetchUsers() {
  try {
    const client = Axon.new();
    const response = await client.get('https://api.example.com/users');
    console.log(response.data);
  } catch (error) {
    console.error('Failed to fetch users:', error);
  }
}

fetchUsers();
```

### 2. 認証とカスタムヘッダー

ここからが`Axon`の真骨頂です。メソッドチェーンを使って、リクエストに必要な設定を順番に追加していきます。

```typescript
async function postArticle(title: string, content: string) {
  const authToken = 'your_super_secret_token';

  // メソッドを繋げてリクエストを構築
  const response = await Axon.new()
    .bearer(authToken) // Bearerトークンを追加
    .setHeader('X-Request-ID', 'unique-request-id-123') // カスタムヘッダーを追加
    .post('https://api.example.com/articles', { title, content });

  console.log('Article created:', response.data);
}
```

`axios`で同じことをしようとすると設定オブジェクトを別途作成する必要がありますが、`Axon`なら一連の処理として記述でき、コードの可読性が劇的に向上します。

# `Axon`の舞台裏：コードで見る設計思想

`Axon`の挙動を支える、いくつかの重要な実装を見ていきましょう。

### 1. クラスの実装

```ts:Axon.ts
import https from "https";
import FormData from "form-data";
import axios from "axios";
import * as _ from "axios";

interface Options {
  /**
   * trueに設定すると、自己署名証明書など、信頼性のないTLS証明書を許可します。
   * 開発環境でのみ使用してください。
   */
  dangerouslyAllowInsecure?: boolean;
}

/**
 * axiosをラップする、イミュータブルなAPIクライアントビルダー
 */
class Axon {
  private readonly instance: _.AxiosInstance;
  private readonly config: _.AxiosRequestConfig;

  private constructor(
    config: _.AxiosRequestConfig,
    instance?: _.AxiosInstance
  ) {
    this.config = config;
    this.instance = instance || axios.new();
  }
}

export default Axon;
```

**📌 ポイント**

>* コンストラクタをprivateにすることで、意図しない方法でのインスタンス化を防いでいる。
>* `instance`と`config`はインスタンスそれぞれ持っているので、互いに干渉しない。

### 2. インスタンスの作成

`Axon`は`new Axon()`で直接インスタンス化するのではなく、`Axon.new()`という静的メソッドから生成します。これにより、インスタンス化前の初期設定（httpsエージェントの差し替えなど）を安全に行えます。

```ts:Axon.ts
public static new(options?: Options): Axon {
  const config: _.AxiosRequestConfig = {};

  if (options?.dangerouslyAllowInsecure) {
    console.warn(
      "⚠️ 警告: TLS証明書の検証が無効になっています。本番環境では絶対に使用しないでください。"
    );
    config.httpsAgent = new https.Agent({
      rejectUnauthorized: false,
    });
  }

  return new Axon(config);
}
```

**📌 ポイント**

>* httpsエージェントを通じて、TLSの無効化が可能

### 3. CRUDの実装

```ts:Axon.ts
public async get(url: string): Promise<_.AxiosResponse> {
  return this.instance.get(url, this.config);
}
```
```ts:Axon.ts
public async post(
  url: string,
  payload: object | FormData
): Promise<_.AxiosResponse> {
  return this.instance.post(url, payload, this.config);
}
```
```ts:Axon.ts
public async put(url: string, payload: object): Promise<_.AxiosResponse> {
  return this.instance.put(url, payload, this.config);
}
```
```ts:Axon.ts
public async delete(url: string): Promise<_.AxiosResponse> {
  return this.instance.delete(url, this.config);
}
```

**📌 ポイント**

>* 後述のチェーンメソッドを組み合わせれば、多様なリクエストが可能

### 4. チェーンメソッドの実装

各チェーンメソッドは、現在の設定（`this.config`）と新しい設定をマージし、それを元に新しい`Axon`インスタンスを返します。これがイミュータブルな設計の核です。

```ts:Axon.ts
public setHeader(key: string, value: string): Axon {
  const newConfig = {
    ...this.config,
    headers: { ...this.config.headers, [key]: value },
  };
  return new Axon(newConfig, this.instance);
}
```
```ts:Axon.ts
public params(params: object): Axon {
  const newConfig = { ...this.config, params };
  return new Axon(newConfig, this.instance);
}
```
```ts:Axon.ts
public basic(token: string): Axon {
  return this.setHeader("Authorization", `Basic ${token}`);
}
```
```ts:Axon.ts
public bearer(token: string): Axon {
  return this.setHeader("Authorization", `Bearer ${token}`);
}
```
```ts:Axon.ts
public encodeUrl(): Axon {
  return this.setHeader("Content-Type", "application/x-www-form-urlencoded");
}
```
```ts:Axon.ts
public octet(): Axon {
  return this.setHeader("Content-Type", "application/octet-stream");
}
```
```ts:Axon.ts
public length(contentLength: number): Axon {
  return this.setHeader("Content-Length", String(contentLength));
}
```
```ts:Axon.ts
public digest(digest: string): Axon {
  return this.setHeader("Digest", `sha-256=${digest}`);
}
```
```ts:Axon.ts
public range(offset: number, end: number, fileSize: number): Axon {
  return this.setHeader(
    "Content-Range",
    `bytes ${offset}-${end - 1}/${fileSize}`
  );
}
```
```ts:Axon.ts
public transformRequest(
  transformers: _.AxiosRequestTransformer | _.AxiosRequestTransformer[]
): Axon {
  const newConfig = { ...this.config, transformRequest: transformers };
  return new Axon(newConfig, this.instance);
}
```
```ts:Axon.ts
public responseType(responseType: _.ResponseType): Axon {
  const newConfig = { ...this.config, responseType };
  return new Axon(newConfig, this.instance);
}
```

**📌 ポイント**

>* 常に`new Axon(...)`を返すことで、元のインスタンスの状態を決して汚染しない。
>* `Content-Type`はデフォルトで`json`だが、設定が可能
>* `Authorization`は`Basic`と`Bearer`の２種類をサポート

# Axonの特徴

### 1. イミュータブル設計の力

`Axon`の最大の特徴は **イミュータブル（不変性）** です。`.bearer()` や `.setHeader()`のようなメソッドを呼び出すたびに、元のインスタンスが変更されるのではなく、**新しい設定を持った新しいインスタンスが返されます。**

これにより、ベースとなるクライアント設定を安全に再利用できます。

```typescript
async function performUserActions() {
  // 1. ベースとなるクライアントを作成
  const baseClient = Axon.new()
    .setHeader('X-App-Version', '1.0.0');

  // 2. ユーザーA用のクライアントをベースから作成
  const clientForUserA = baseClient.bearer('token_for_user_a');
  const userAData = await clientForUserA.get('https://api.example.com/me');
  console.log('User A:', userAData.data.name);

  // 3. ユーザーB用のクライアントをベースから作成
  const clientForUserB = baseClient.bearer('token_for_user_b');
  const userBData = await clientForUserB.get('https://api.example.com/me');
  console.log('User B:', userBData.data.name);

  // clientForUserAの認証設定がbaseClientやclientForUserBに影響することはない！
}
```

もし`Axon`がイミュータブルでなければ、`clientForUserA`で設定したトークンが、後の`clientForUserB`の通信でも使われてしまう、という深刻なバグを引き起こす可能性がありました。イミュータブルな設計は、このような副作用を未然に防ぎます。

### 2. 開発時の柔軟性

もちろん、開発時には柔軟性も必要です。`Axon`は、自己署名証明書を使うローカルサーバーと通信するために、証明書の検証を無効にするオプションも提供しています。

```typescript
// 'development'環境でのみ危険なオプションを有効化する
const isDevelopment = process.env.NODE_ENV === 'development';

const devClient = Axon.new({
  dangerouslyAllowInsecure: isDevelopment,
});

// これでローカルのHTTPSサーバーとも通信できる
// await devClient.get('https://localhost:3001/test');
```

このオプションは、その名の通り「危険」であることが明確に示されており、開発者が意図を持って使うように設計されています。**デフォルトは常に安全**、これが`Axon`の哲学です。

**補足：なぜSSLを無視する？**

https://qiita.com/oharu121/items/49504ea4a6126936431d

# まとめ

`Axon`は、単なる`axios`のラッパーではありません。**Factory Method**、**Builder**、そして**Immutability**といったデザインパターンを組み合わせることで、宣言的で、安全で、そして使っていて楽しいAPIクライアント体験を提供します。

日々のAPI設定の煩わしさから解放されたい方は、ぜひ一度、`Axon`のようなアプローチを試してみてはいかがでしょうか。あなたのコードベースが、よりクリーンで堅牢なものになるはずです。