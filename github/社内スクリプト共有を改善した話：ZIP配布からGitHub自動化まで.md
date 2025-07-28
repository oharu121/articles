# 背景と課題

私のチームでは、業務効率化のためにNode.js製の自作スクリプトをメンバー間で共有していました。
当初は、**ZIPファイルをフォルダにアップロードし、手動で展開して使ってもらう**という、原始的な運用を行っていました。

しかし、次第に以下のような課題が浮き彫りになりました：

>* **更新ごとにZIP作成・アップロードが必要**で、運用コストが高い
>* **最新版が行き渡るまで時間差が発生**し、バージョンずれによる不具合が多発
>* **手動展開での上書きミスが頻発**し、導入にストレスが伴う
>* **どのバージョンで不具合が出たのか追跡困難**

**🏰 GitHub活用に立ちはだかる壁**

会社のセキュリティ要件により、**GitHubへのアクセスはPomeriumを経由した社内トンネル接続が必須**でした。そのため、CLIベースでのGit操作やスクリプト自動化に大きな制約がありました。

> → 解決策：GitHub管理とトンネル接続の自動化

この状況を改善するために、私は **GitHubによるスクリプト一元管理の仕組み** を提案・実装しました。

# 1. Pomeriumトンネルの自動生成

まずは**Pomeriumバイナリを使って、OSに依存せずトンネルを自動確立**できる仕組みを実装しました。

```ts:Setup.ts
public initPomerium(): Promise<void> {
  const pomeriumPath = environment.CURRENT_OS === "win32"
    ? paths.POMERIUM_PATH_WIN
    : paths.POMERIUM_PATH_MAC;

  return new Promise((resolve, reject) => {
    const proc = spawn(pomeriumPath, ["tcp", pomeriumHost, "--listen", `:${pomeriumPort}`]);

    proc.stderr.on("data", (chunk) => {
      const msg = chunk.toString();
      if (msg.includes("listening on")) {
        Logger.success(`Pomerium is listening on localhost:${pomeriumPort}`);
        resolve();
      } else {
        reject(new Error("Failed to start Pomerium tunnel"));
      }
    });
  });
}
```

# 2. Gitのプロキシ設定とCredentialの自動登録

トンネルが確立されたら、Git操作がトンネル経由でGitHubに届くようにプロキシや認証情報を自動設定します。

```ts:Setup.ts
await exec("git config http.proxy 127.0.0.1:8080");
await exec("git config --global user.name xxx");
await exec("git config --global user.email xxx@example.com");
await exec("git config --global credential.http://github.company.internal.provider github");
```

# 3. forwardLocalPort：トンネル処理の再利用可能な関数化

GitHubアクセス前にトンネルを毎回確立するため、共通メソッドとして抽出・整理しました。

```ts:Setup.ts
public forwardLocalPort(): Promise<void> {
  const pomeriumPath = environment.CURRENT_OS === "win32"
    ? paths.POMERIUM_PATH_WIN
    : paths.POMERIUM_PATH_MAC;
  
  return new Promise((resolve) => {
    const tunnel = spawn("ssh", [
      "-o", `ProxyCommand=${pomeriumPath} tcp --listen - %h:%p`,
      "-o", "StrictHostKeyChecking=no",
      "-L", "8080:localhost:80",
      hosts.REMOTE_DOMAIN_STG,
    ]);

    tunnel.stderr.on("data", (data) => {
      const msg = data.toString();
      if (msg.includes("connection established")) {
        Logger.success("Tunnel is ready on localhost:8080");
        resolve();
      }
    });
  });
}
```

**📌 ポイント**

>* このメソッドを `clone`・`setup`・`pull` など全ての操作前に呼び出すことで、GitHubへの接続を自動で行えるようにしました。


# 4. 「npm run install」一発で環境構築

非エンジニアのメンバーでも簡単にセットアップできるよう、初期化処理を一括化しました。

```ts:Setup.ts
public async setup(): Promise<void> {
  await this.forwardLocalPort();
  await exec("git init");
  await exec("git config http.proxy 127.0.0.1:8080");
  await exec("git remote add origin http://<user>:<token>@repo.url");
  await exec("git fetch origin");
  await exec("git reset --hard origin/main");
  await exec("npm install");
}
```

**📌 ポイント**

>* 今では、**「npm run install」コマンドを叩くだけで、誰でも数秒で最新スクリプトを導入可能**になりました。

# おわりに

**成果とまとめ**

>* ✅ **ZIP配布からの脱却**：手動配布が不要になり、運用コストがゼロに
>* ✅ **バージョン管理の明確化**：リリースログで変更履歴が一目瞭然
>* ✅ **導入が簡単に**：非エンジニアでも「npm run install」で完了
>* ✅ **セキュリティ制約の克服**：Pomeriumトンネルを自動で確立し、GitHub連携も自動化
>* ✅ **DXの向上**：手間や混乱を排除し、開発者体験が劇的に改善

この取り組みは、単なるスクリプト共有の改善にとどまらず、**チームの開発体験（DX）を大きく底上げする一歩**となりました。