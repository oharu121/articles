## SSH 鍵の生成と使用ガイド（Windows 対応）

## はじめに

リモートサーバへ SSH 接続するたびにパスワードを入力するのは、面倒で非効率です。
そこで役立つのが `ssh-keygen`。これを使うことで、

> - パスワード入力を不要にできる
> - セキュリティも向上する
> - 自動化やスクリプト運用もスムーズになる

というメリットがあります。この記事では、**Windows** および **Linux（ジャンプサーバ）** 環境での使い方を解説します。

## Windows 環境

#### 1. `.ssh` フォルダの作成（初回のみ）

まず、SSH 鍵を保存する専用フォルダを作成します（存在していればスキップ可）：

```batch
mkdir "C:\Users\YourUsername\.ssh"
```

#### 2. SSH 鍵の生成

次のコマンドを実行すると、RSA 4096bit の鍵ペアを作成できます：

```batch
ssh-keygen -t rsa -b 4096 -f "C:\Users\YourUsername\.ssh\id_rsa" -N ""
```

**📌 ポイント**

> - `-t rsa`: 鍵の種類（RSA）
> - `-b 4096`: ビット長（セキュアな 4096bit）
> - `-f`: 鍵の保存先
> - `-N ""`: パスフレーズなし

実行すると以下のようなメッセージが表示されれば成功です：

```cmd
Generating public/private rsa key pair.
Your identification has been saved in C:\Users\YourUsername\.ssh\id_rsa
Your public key has been saved in C:\Users\YourUsername\.ssh\id_rsa.pub
The key fingerprint is:
SHA256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX YourUsername@YourComputer
The key's randomart image is:
+---[RSA 4096]----+
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
+----[SHA256]-----+
```

#### 3. 公開鍵をサーバにアップロード

SSH 鍵の秘密鍵と公開鍵が生成されたら、リモートサーバに SSH 鍵の公開鍵をアップロードします。PowerShell を開いて、以下のコマンドを実行します：

```powershell
type $env:USERPROFILE\.ssh\id_rsa.pub | ssh [username]@[remote_host] "cat >> .ssh/authorized_keys"
```

> 🔒 この操作の際、リモートサーバのパスワードを一度だけ入力する必要があります。

## ジャンプサーバ環境（Linux）

もし踏み台（ジャンプ）サーバ経由で最終サーバにアクセスする場合は、ジャンプサーバ上でも同様の設定が必要です。

#### 1. SSH 鍵の生成

```
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

#### 2. 公開鍵をリモートサーバへコピー

```bash
ssh-copy-id [sshユーザ名]@[リモートサーバのホスト or IP]
```

> 💡 `ssh-copy-id` は Linux/macOS に標準搭載されていますが、Windows にはありません。ジャンプサーバなど Linux 環境から実行してください。

## まとめ

SSH 鍵の生成と使用には、以下のようなメリットがあります：

> 1.  セキュリティの向上：強力な暗号化により、パスワード認証よりも安全です
> 2.  利便性の向上：パスワード入力が不要になり、ログインプロセスが簡素化されます
> 3.  管理の容易さ：中央集中型の鍵管理が可能になります
> 4.  自動化の促進：スクリプトやツールによる自動ログインが容易になります
> 5.  鍵の一元管理：一つの秘密鍵で複数サーバを管理可能

SSH 鍵認証は一度設定すれば、**快適かつ安全にサーバ操作を行える** 非常に強力な手段です。`ssh-keygen` を活用して、毎日の開発や運用をもっとスムーズにしましょう。
