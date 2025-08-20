## SSH 経由のリモート操作を自動化＆安定化させる仕組みを作った話

## はじめに

Jump サーバーを経由して Remote サーバーに接続し、そこから**コマンド実行**や**ファイル操作**、**Kubernetes 設定**を行う業務が増えてきました。

しかし、実運用では下記のような課題があります：

- 接続が不安定（途中で切れる）
- バッチ処理の途中で止まると復旧が面倒
- ファイルや設定の環境差異でミスが発生

そこで私は、**TypeScript + SSH2 ライブラリ**を使って、
Jump 経由の Remote 接続をラップした安定化＆自動復活フレームワークを作りました。

**🖼️ システム構成図**

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/ae9a1696-85d3-4ddf-a6fe-8a0bc5c95613.png)

## Remote クラス：リモート通信の司令塔

リモートへの SSH 接続、コマンド実行、SFTP 転送など、**すべてのリモート操作を 1 クラスに集約**しています。

**📦 主な責務**

> - Jump サーバー接続・ポートフォワーディング
> - リモートサーバー接続・コマンド実行
> - ファイル送受信（SFTP）
> - 公開鍵の自動生成・登録
> - シェルストリームの操作

**💡 ハイライト**

> - `initJumpTunnel()`：Jump サーバーと接続＋鍵確認
> - `fowardToRemote()`：Remote サーバーへ SSH ポートフォワーディング
> - `uploadFileToRemote()`：ローカル →Remote へのファイル転送
> - `execRemote(cmd)`：リモートで任意コマンド実行
> - `openShell()`：インタラクティブシェルを開く
> - `waitForString(stream, text)`：シェル出力から特定文字を待機

**🛠 安定化への工夫**

> - SSH 秘密鍵がなければ自動生成＋ package.json 更新
> - 再接続時はセッションを自動復旧
> - `SessionDB` により、prod/stg などの環境切替も安全に抽象化

#### 1. クラス定義

```typescript:Remote.ts
class Remote {
  private jumpTunnel?: Client;
  private remoteTunnel?: Client;
  private jumpSFTP?: SFTP;
  private remoteSFTP?: SFTP;
}

export default new Remote();
```

**📌 ポイント**

> - `jumpTunnel`: Jump サーバーとの SSH クライアントインスタンス
> - `remoteTunnel`: リモートサーバーへの SSH クライアントインスタンス
> - `jumpSFTP`, `remoteSFTP`: それぞれの SFTP 用オブジェクト
> - Remote クラスを **インスタンス化して即エクスポート**
>   → どこでも `import Remote from './Remote'` で使える！

#### 2. 接続情報取得 (環境に応じて切り替え)

- Pomerium ポート番号取得

```typescript:Remote.ts
private pomeriumPort(): number {
  const env = SessionDB.getEnv();
  return env === "prod" ? ports.POMERIUM_PORT_PROD : ports.POMERIUM_PORT_STG;
}
```

- リモートホスト名取得

```typescript:Remote.ts
private remoteHost(): string {
  const env = SessionDB.getEnv();
  return env === "prod" ? hosts.REMOTE_HOST_PROD : hosts.REMOTE_HOST_STG;
}
```

- SSH ユーザー名取得

```typescript:Remote.ts
private remoteUsername(): string {
  const env = SessionDB.getEnv();
  return env === "prod" ? credentials.SSH_USERNAME_PROD : credentials.SSH_USERNAME_STG;
}
```

- namespace 取得

```typescript:Remote.ts
public static namespace(): string {
  const env = SessionDB.getEnv();
  switch (env) {
    case "stg":
      return names.NAMESPACE_STG;
    case "prod":
      return names.NAMESPACE_PROD;
    default:
      return "";
  }
}
```

- SSH プロファイル取得

```typescript:Remote.ts
public static getSSHProfile(env: Env): ConnectConfig {
  const env = SessionDB.getEnv();
  switch (env) {
    case "prod":
      return {
        host: "127.0.0.1",
        port: ports.POMERIUM_PORT_PROD,
        username: credentials.SSH_USERNAME_PROD,
        password: credentials.JUMP_SERVER_PASSWORD_PROD,
        readyTimeout: 60000,
      };
    default:
      return {
        host: "127.0.0.1",
        port: ports.POMERIUM_PORT_STG,
        username: credentials.SSH_USERNAME_STG,
        password: credentials.JUMP_SERVER_PASSWORD_STG,
        readyTimeout: 60000,
      };
  }
}
```

#### 3. トンネル初期化

- Jump トンネル初期化

```typescript:Remote.ts
private async initJumpTunnel() {
  await this.connectJumpServer();
  await this.checkPublicKey();
}
```

- Remote トンネル初期化

```typescript:Remote.ts
private async initRemoteTunnel() {
  await this.fowardToRemote();
  await Robin.start();
}
```

**📌 ポイント**

> - **Jump サーバー接続**と**公開鍵登録チェック**を実施
> - Jump サーバー経由でリモート接続＋ Robin 起動

#### 4. 開鍵認証セットアップ

```typescript:Remote.ts
private async checkPublicKey() {
  if (!packageJson.pubKeyEnabled) {
    await this.getPrivateKey();
    await this.checkAuthorizedKeys();
  }
}
```

**📌 ポイント**

> - まだ鍵認証が有効でないなら、秘密鍵作成＋公開鍵登録をする。

#### 5. Jump サーバー接続処理

```typescript:Remote.ts
public async connectJumpServer(): Promise<void> {
  await CommandLine.initPomerium();
  return new Promise((resolve, reject) => {
    const jumpTunnel = new Client();
    jumpTunnel
      .on("ready", () => {
        Logger.success(`Connected to jump server on port ${this.pomeriumPort()}`);
        this.jumpTunnel = jumpTunnel;
        resolve();
      })
      .on("error", (err) => {
        Logger.error(`Failed to connect to port ${this.pomeriumPort()} locally.`);
        reject(err);
      })
      .connect(this.sshProfile());
  });
}
```

**📌 ポイント**

> - Pomerium ポートに接続 → 成功したら jumpTunnel 保存

#### 6. リモートサーバーへのポートフォワード接続

```typescript:Remote.ts
public async fowardToRemote(): Promise<void> {
  if (!this.jumpTunnel) await this.initJumpTunnel();
  return new Promise((resolve, reject) => {
    if (!this.jumpTunnel) throw new Error();
    this.jumpTunnel.forwardOut(
      "127.0.0.1",
      8000,
      this.remoteHost(),
      22,
      (err, stream) => {
        if (err) reject(err);
        const remoteTunnel = new Client();
        remoteTunnel
          .on("ready", () => {
            Logger.success(`forwarded out to ${this.remoteHost()}`);
            this.remoteTunnel = remoteTunnel;
            resolve();
          })
          .on("error", (err) => reject(err))
          .connect({
            sock: stream,
            ...this.sshProfile(),
          } as ConnectConfig);
      }
    );
  });
}
```

**📌 ポイント**

> - Jump サーバーを**中継**して**リモートサーバー 22 番ポートに forwardOut**する！

#### 7. SFTP 初期化

- Jump 用

```typescript:Remote.ts
private async initJumpSFTP() {
  if (!this.jumpTunnel) await this.initJumpTunnel();
  if (!this.jumpTunnel) throw new Error();
  this.jumpSFTP = new SFTP(this.jumpTunnel);
}
```

- Remote 用

```typescript:Remote.ts
private async initRemoteSFTP() {
  if (!this.remoteTunnel) await this.initRemoteTunnel();
  if (!this.remoteTunnel) throw new Error();
  this.remoteSFTP = new SFTP(this.remoteTunnel);
}
```

**📌 ポイント**

> - SSH 接続セッションを渡して、SFTP 通信できるように初期化

#### 8. ファイルアップロード

```typescript
public async uploadFileToRemote(fileName: string): Promise<void> {
  if (!this.remoteSFTP) await this.initRemoteSFTP();
  const localPath = `${paths.LOCAL_File_DIR}/${fileName}`;
  const remotePath = `${paths.REMOTE_File_DIR}/${fileName}`;
  await this.remoteSFTP?.fastput(localPath, remotePath);
}
```

**📌 ポイント**

> : ローカルからリモートへ**ファイル転送**します。

#### 9. SSH 秘密鍵の取得・生成 `getPrivateKey`

```typescript:Remote.ts
public static async getPrivateKey(): Promise<string | undefined> {
  let privateKey;
  let isPrivateKeyExisted;
  const sshDir = `C:\\Users\\${environment.OS_USERNAME}\\.ssh`;

  if (!fs.existsSync(sshDir)) {
    Logger.warning(`.ssh folder does not exist for user ${environment.OS_USERNAME}`);
    Logger.task(`Creating ${sshDir}...`);
    fs.mkdirSync(sshDir, { recursive: true });
  }

  try {
    privateKey = fs.readFileSync(`${sshDir}\\id_rsa`, "utf8");
    Logger.success("found private key locally.");
    isPrivateKeyExisted = true;
  } catch (err) {
    Logger.warning("Private key not found.");
    Logger.task(`Generating new SSH key pair for user ${environment.OS_USERNAME}...`);
    isPrivateKeyExisted = false;
  }

  if (!isPrivateKeyExisted) {
    packageJson.pubKeyEnabled = false;
    const updatedPackageJson = JSON.stringify(packageJson, null, 2);
    fs.writeFileSync("./package.json", updatedPackageJson, "utf-8");

    try {
      await exec(
        `ssh-keygen -t rsa -b 4096 -f C:\\Users\\${environment.OS_USERNAME}\\.ssh\\id_rsa -q -N ""`
      );
      Logger.success("SSH key pair generated successfully.");
    } catch (error) {
      Logger.error(`exec error: ${error}`);
    }
  }

  return privateKey;
}
```

**📌 ポイント**

> - `.ssh`ディレクトリがなければ作成
> - 秘密鍵がなければ `ssh-keygen` で作成
> - 作った場合は `package.json`を更新してフラグを保存

#### 10. リモートコマンド実行

- jump 用

```typescript:Remote.ts
public async execJump(cmd: string, debug: boolean = false): Promise<string> {
  if (!this.jumpTunnel) await this.initJumpTunnel();

  return new Promise((resolve, reject) => {
    if (!this.jumpTunnel) throw new Error();
    this.jumpTunnel.exec(cmd, (err, stream) => {
      if (err) reject(err);

      let data = "";
      let meta = "";
      stream.on("data", (chunk: Buffer) => (data += chunk.toString()));
      stream.stderr.on("data", (chunk: Buffer) => (meta += chunk.toString()));
      stream.on("end", () => {
        if (debug) console.log(meta);
        resolve(data);
      });
    });
  });
}
```

**📌 ポイント**

> - ジャンプサーバーでコマンドを実行して、その結果 (`stdout`) を返す。
> - `stderr`はデバッグオプションで表示できる。

- remote 用

```typescript:Remote.ts
public async execRemote(cmd: string, debug: boolean = false): Promise<string> {
  if (!this.remoteTunnel) await this.initRemoteTunnel();

  return new Promise((resolve, reject) => {
    if (!this.remoteTunnel) throw new Error();
    this.remoteTunnel.exec(cmd, (err, stream) => {
      if (err) reject(err);

      let data = "";
      let meta = "";
      stream.on("data", (chunk: Buffer) => (data += chunk.toString()));
      stream.stderr.on("data", (chunk: Buffer) => (meta += chunk.toString()));
      stream.on("end", () => {
        if (debug) console.log(meta);
        resolve(data);
      });
    });
  });
}
```

**📌 ポイント**

> - リモートサーバーでコマンドを実行して、その結果 (`stdout`) を返す。
> - `stderr`はデバッグオプションで表示できる。

#### 11. シェルセッションを開く

```typescript:Remote.ts
public openShell(): Promise<ClientChannel> {
  if (!this.remoteTunnel) await this.initRemoteTunnel();

  return new Promise((resolve, reject) => {
    if (!this.remoteTunnel) throw new Error();
    this.remoteTunnel.shell((err, stream) => {
      if (err) reject(err);
      else resolve(stream);
    });
  });
}
```

**📌 ポイント**

> - 対話式のシェルストリーム (`ClientChannel`) を開く。
> - ログイン後にコマンドを打てるようにするため。

#### 12. ストリームから特定文字列を待つ

```typescript:Remote.ts
public async waitForString(
  stream: ClientChannel,
  expectedString: string
): Promise<string> {
  return new Promise((resolve, reject) => {
    let data = "";
    stream
      .on("data", (chunk: Buffer) => {
        Logger.info(chunk.toString());
        data += chunk.toString();
        if (data.includes(expectedString)) {
          resolve(data);
        }
      })
      .stderr.on("data", (data: Buffer) => {
        reject(new Error(`STDERR: ${data}`));
      })
      .on("close", () => {
        reject(new Error("unexpected stream close"));
      });
  });
}
```

**📌 ポイント**

> - シェル通信中、特定のキーワードが現れるのを待つ。
> - 例えば「ログイン成功」の文字列を待つ用途など。

## Robin クラス：Kubernetes の認証＆コンテキスト操作

Jump→Remote 接続後に、`kubectl`を操作する前提で、Kubernetes の文脈も整えています。

**📦 主な処理**

> - 古い Robin コンテキスト削除
> - Cluster VIP/namespace/tenant に応じた新しい設定適用
> - `robin login`と`kubectl config`を自動で実行

**💡 利点**

> - `SessionDB`の環境情報から自動切替
> - マニュアル操作ゼロで k8s 環境にログイン・接続可能

#### 1. クラス定義

```typescript:Robin.ts
import Remote form "./Remote";
class Robin {}

export default new Robin();
```

**📌 ポイント**

> - 前述の `Remote` クラスをインポートして、その `execRemote` を使う

#### 2. 接続情報取得 (環境に応じて切り替え)

- Namespace 取得

```typescript:Robin.ts
private namespace(): string {
  const env = SessionDB.getEnv();
  switch(env){
    case"stg1":
      return names.NAMESPACE_STG1;
    case"prod":
      return names.NAMESPACE_PROD;
    default:
      return"";
  }
}
```

- tenant 取得

```typescript:Robin.ts
private tenant(): string {
  const env = SessionDB.getEnv();
  switch (env) {
    case "stg":
      return names.TENANT_STG;
    case "prod":
      return names.TENANT_PROD;
  }
}
```

- cluster vip 取得

```typescript:Robin.ts
private clusterVIP(): string {
  const env = SessionDB.getEnv();
  switch (env) {
    case "stg":
      return hosts.CLUSTER_VIP_STG;
    case "prod":
      return hosts.CLUSTER_VIP_PROD;
  }
}
```

#### 3. コンテキスト編集

- 古い Robin コンテキスト削除

```typescript:Robin.ts
private async removeOldContext(): Promise<void> {
  await Remote.execRemote("rm -rf ~/.robin");
}
```

**📌 ポイント**

> - リモートサーバー上の`~/.robin`ディレクトリを削除する。
> - 新しい Robin コンテキスト追加

```typescript:Robin.ts
private async addNewContext(): Promise<void> {
  const cmd = [
    `robin client`,
    `add-context ${this.clusterVIP}`,
    `--port ${ports.ROBIN_PORT}`,
    `--event-port ${ports.ROBIN_PORT}`,
    `--file-port ${ports.ROBIN_PORT}`,
    `--set-current --product platform`,
  ].join("");

  await Remote.execRemote(cmd).then((result) => {
    if (typeof result === "string") Logger.info(result);
  });
}
```

**📌 ポイント**

> - Robin クライアントで新しいコンテキストを作成
> - Cluster VIP やポート番号を使って設定
> - 成功したらログ出力

#### 4. ログイン

```typescript:Robin.ts
private async login(): Promise<void> {
  const cmd = [
    `robin login ${credentials.ROBIN_USERNAME_STG}`,
    `--password ${credentials.ROBIN_PASSWORD_STG}`,
    `--tenant ${this.tenant}`,
  ].join("");

  const robinLoginResult = await Remote.execRemote(cmd);

  if (typeof robinLoginResult === "string") {
    if (robinLoginResult.includes("is logged into")) {
      Logger.success(`Logged into Robin with ${this.tenant}`);
      Logger.info(robinLoginResult);
    } else {
      Logger.error("user can't login to robin");
      Logger.info(robinLoginResult);
      process.exit(1);
    }
  }
}
```

**📌 ポイント**

> - `robin login`コマンドを実行
> - 成功したらログ、失敗したらエラーログ＋プロセス終了

#### 5. kubeconfig ファイル保存

```typescript:Robin.ts
private async saveKubeConfig(): Promise<void> {
  const cmd = [
    `mkdir -p ~/.kube`,
    `&&`,
    `robin k8s get-kube-config`,
    `--save-as-file`,
    `--dest-dir ~/.kube`,
  ].join("");

  await Remote.execRemote(cmd);
}
```

**📌 ポイント**

> - リモート上に`.kube`フォルダを作成
> - Robin から k8s 設定ファイルを保存

#### 6. 現在の k8s コンテキストをセット

```typescript:Robin.ts
private async setCurrentContext(): Promise<void> {
  const cmd = [
    `kubectl config`,
    `set-context`,
    `--current`,
    `--namespace=${this.namespace}`,
  ].join("");

  await Remote.execRemote(cmd).then((result) => {
    if (typeof result === "string") Logger.info(result);
  });
}
```

**📌 ポイント**

> - `kubectl config set-context`で、今使う namespace を指定する。

## SFTP クラス：安全なファイル送受信のラッパー

SSH トンネル上でのファイル操作をまとめています。

**📦 機能**

- `fastput()`：ローカル → リモートへアップロード
- `fastget()`：リモート → ローカルへダウンロード
- `appendFile()`：ログや結果の追記書き込み
- `readDir()`：リモートディレクトリ一覧取得

#### 1. クラス定義

```typescript:SFTP.ts
class SFTP {
  private tunnel: Client;
  private sftpTunnel?: SFTPWrapper;
```

**📌 ポイント**

> - `tunnel`：SSH 接続しているクライアント
> - `sftpTunnel`：SFTP 用セッション（未初期化なら接続する）

#### 2. コンストラクタ

```typescript:SFTP.ts
constructor(tunnel: Client) {
  this.tunnel = tunnel;
}
```

**📌 ポイント**

> - 外から渡された SSH クライアント (`tunnel`) を内部に保持する

#### 3. SFTP セッション初期化

```typescript:SFTP.ts
private initSftp(): Promise<void> {
  return new Promise((resolve, reject) => {
    this.tunnel.sftp((err, sftp) => {
      if (err) reject(err);
      this.sftpTunnel = sftp;
      resolve();
    });
  });
}
```

**📌 ポイント**

> - SSH トンネル (`tunnel`) を使って新たに SFTP セッションを開始する
> - 失敗したらエラー

#### 4. ファイルアップロード

```typescript:SFTP.ts
public async fastput(localPath: string, remotePath: string): Promise<boolean> {
  if (!this.sftpTunnel) await this.initSftp();

  await this.createIfNotExisted(path.dirname(remotePath));
  return new Promise((resolve) => {
    if (!this.sftpTunnel) throw new Error();

    this.sftpTunnel.fastPut(localPath, remotePath, {}, (err) => {
      if (err) resolve(false);
      else resolve(true);
    });
  });
}
```

**📌 ポイント**

> - SFTP セッションがなければ初期化
> - 先にアップロード先のディレクトリが存在するか確認・作成
> - `fastPut`でローカルファイルをリモートにアップロード

#### 5. ファイルダウンロード

```typescript:SFTP.ts
public async fastget(localPath: string, remotePath: string): Promise<boolean> {
  if (!this.sftpTunnel) await this.initSftp();

  return new Promise((resolve) => {
    if (!this.sftpTunnel) throw new Error();

    this.sftpTunnel.fastGet(remotePath, localPath, (err) => {
      if (err) resolve(false);
      else resolve(true);
    });
  });
}
```

**📌 ポイント**

> - リモートのファイルをローカルにダウンロードする

#### 6. ディレクトリ存在チェック

```typescript:SFTP.ts
public async checkDir(remotePath: string): Promise<boolean> {
  if (!this.sftpTunnel) await this.initSftp();

  return new Promise((resolve) => {
    if (!this.sftpTunnel) throw new Error();

    this.sftpTunnel.stat(remotePath, (err) => {
      if (err) resolve(false);
      else resolve(true);
    });
  });
}
```

**📌 ポイント**

> - 指定されたパスがリモートに存在するかチェックする

#### 7. ディレクトリ作成

```typescript:SFTP.ts
public async makeDir(remotePath: string): Promise<boolean> {
  if (!this.sftpTunnel) await this.initSftp();

  return new Promise((resolve) => {
    if (!this.sftpTunnel) throw new Error();

    this.sftpTunnel.mkdir(remotePath, (err) => {
      if (err) {
        console.log(err);
        resolve(false);
      } else {
        resolve(true);
      }
    });
  });
}
```

**📌 ポイント**

- 指定されたパスにリモートでディレクトリを作成する

#### 8. ファイル追記

```typescript:SFTP.ts
public async appendFile(remotePath: string, text: string): Promise<void> {
  if (!this.sftpTunnel) await this.initSftp();

  return new Promise((resolve, reject) => {
    if (!this.sftpTunnel) throw new Error();

    this.sftpTunnel.appendFile(remotePath, text, (err) => {
      if (err) reject(err);
      else resolve();
    });
  });
}
```

**📌 ポイント**

> - 指定ファイルにテキストを追記する（なければエラー）

#### 9. ディレクトリ一覧取得

```typescript:SFTP.ts
public async readDir(path: string): Promise<FileEntry[]> {
  if (!this.sftpTunnel) await this.initSftp();

  return new Promise((resolve, reject) => {
    if (!this.sftpTunnel) throw new Error();

    this.sftpTunnel.readdir(path, (err, list) => {
      if (err) reject(err);
      else resolve(list);
    });
  });
}
```

**📌 ポイント**

- 指定パスのディレクトリの中身一覧を取得する

#### 10. パス存在チェック＆作成

```typescript:SFTP.ts
public async createIfNotExisted(pathName: string): Promise<void> {
  const isPathExisted = await this.checkDir(pathName);
  if (!isPathExisted) {
    Logger.warning(`Cannot found ${pathName} for user ${credentials.SSH_USERNAME_STG}`);
    Logger.task(`Trying to create ${pathName} for user ${credentials.SSH_USERNAME_STG}`);
    const pathCreated = await this.makeDir(pathName);

    if (!pathCreated) {
      Logger.error(`Failed to create folder`);
      return;
    }

    Logger.success(`Successfully created remote folder for user ${credentials.SSH_USERNAME_STG}`);
  }
}
```

**📌 ポイント**

- パスがなければ mkdir する
- 成功・失敗ログを出す

## まとめ

Jump サーバーを経由した Remote 操作には以下のような課題がありますが…

- 毎回 SSH コマンドを打つのが面倒
- 環境差異でうっかり事故る
- リモート処理中に接続が切れるとリカバリが難しい

➡ 今回の実装により、

- **秘密鍵の自動生成・登録**
- **SFTP・コマンド・k8s を一気通貫で自動化**
- **安定して再接続＆復旧可能**

が可能になりました。

応用は次の記事をご参照ください！

https://qiita.com/oharu121/items/2cf1f650a89d52fa5abc
