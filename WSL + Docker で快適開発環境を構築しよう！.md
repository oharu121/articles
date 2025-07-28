# WSL + Docker で快適開発環境を構築しよう！

# はじめに

Windows上でLinuxを使いたい、Dockerで開発環境を整えたい──そんな方におすすめなのが **WSL（Windows Subsystem for Linux）** と **Docker Desktop** の連携です。本記事では、WSLとDockerを使って快適な開発環境を構築する方法をわかりやすく解説します。

# 🐧 WSLとは？

WSL（Windows Subsystem for Linux）とは、**Windows上でLinuxのコマンドやアプリケーションを実行できる機能**です。

| バージョン | 特徴                                      |
| ----- | --------------------------------------- |
| WSL 1 | 軽量な互換レイヤー。Linuxカーネルは未使用。                |
| WSL 2 | **本物のLinuxカーネルを搭載**し、パフォーマンスと互換性が大幅に向上。 |

特にWSL 2では、仮想マシンのような重さがなく、**WindowsとLinux間のファイル共有も高速** です。

# 🐳 Dockerと統合するメリット

Docker DesktopはWSL 2と連携することで、以下のようなメリットがあります：

>* Windows上で**Linuxコンテナをシームレスに実行**
>* Dockerボリュームやマウントを**WSLのファイルシステムと連携可能**
>* ネイティブに近いパフォーマンスで開発可能

# WSLの有効化手順

### 1. Windowsの機能の有効化

>1. コントロールパネル →「プログラムと機能」を選択
>2. 左メニューから「Windowsの機能の有効化または無効化」をクリック
>3. 「Windows Subsystem for Linux」にチェックを入れ、「OK」を押下
>4. 再起動

![機能の有効化](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/b0a352ac-7a7a-da8c-7efe-15246e801cc0.png)

---

### 2.Linuxのインストール

>1. Microsoft Storeを開く
>2. 「Ubuntu」など好みのディストリビューションを検索
>3. 「入手」ボタンでインストール
   
![Ubuntuインストール](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/301e0e46-4029-863e-5ff2-db4fb13cb0db.png)

### 3. WSL 2にアップグレード

>1. 管理者権限で PowerShell を開く
>2. WSL 2 をデフォルトバージョンに設定：

```powershell
wsl --set-default-version 2
```

>3. 必要に応じてWSL自体をインストール：

```powershell
wsl --install
```

> 💡 Windows 11 以降は、WSLが標準で統合されているため、`wsl --install`だけで完了することもあります。

# Docker Desktopの設定

### 1. Docker Desktopのインストール

公式サイトからインストーラーを入手し、実行：


https://docs.docker.com/desktop/install/windows-install/

インストール後、PCを再起動してDocker Desktopを起動します。

### 2. WSLとのインテグレーション設定

>1. Docker Desktopの右上の歯車アイコンをクリック
>2. \[Resources] → \[WSL Integration] を選択
>3. 「Enable integration with my default WSL distro」にチェック
>4. 「Ubuntu」などの使用中ディストリビューションもON
>5. 「Apply & Restart」で設定を反映
   
![WSL Integration](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/7aa6ea45-3610-75bf-0827-a5d4a2f76b8e.png)


### 3. Ubuntuを起動して確認

>1. Windowsの検索バーで「ターミナル」と入力し起動
>2. ターミナルのタブから「Ubuntu」を選択
>3. 起動後、下記のように表示されれば成功！

```bash
$ uname -r
5.x.x-... # Linuxカーネルが表示されればOK
```

# まとめ

WSL 2とDockerを組み合わせることで、**Windowsでも本格的なLinux開発環境**を手軽に構築できます。仮想マシンのような重さもなく、**高速かつ柔軟**な開発体験を提供します。

📝 セットアップ完了後は、以下のような開発が可能です：

>* Node.jsやRuby on RailsなどのLinux依存開発
>* VS Codeからの直接操作
>* GitやSSHを使ったLinuxベースのワークフロー