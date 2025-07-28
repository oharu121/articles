# はじめに

企業環境ではセキュリティの観点から、社内ネットワークからのインターネットアクセスが制限されていることがよくあります。

こうした状況でも、リモートサーバを経由してGitHubリポジトリをクローンする方法として、「SSHローカルフォワーディング」が有効です。

この記事では、ローカルフォワーディングを活用してGitHubに安全にアクセスする手順を詳しく解説します。

**前提条件**

>1. ローカルマシンにSSHクライアントがインストールされていること
>1. リモートサーバへのSSHアクセス権限があること
>1. リモートサーバがGitHubにアクセスできること
>1. GitHubアカウントとアクセスしたいリポジトリへの権限があること

# 1. SSHローカルフォワーディングの設定

まず、ローカルマシンからリモートサーバへのSSH接続を確立し、ポートフォワーディングを設定します。

```bash
ssh <username>@<remote-server> -L 8080:localhost:80
```

>usernameはリモートサーバのユーザー名に、remote-serverはリモートサーバのアドレスに置き換えてください。

# 2. GitHubでのトークン作成

右上のプロフィールアイコンをクリックし、Settingsを選択します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/90687b76-953a-b261-097b-8115f297c84c.png)

左サイドメニューからDeveloper settingsをクリックします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/b4ed5ff1-42b7-2508-eb3b-e1e97feb823c.png)

Personal access tokens > Tokens (classic)を選択します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/07414576-c758-bd6c-f6ba-851c8e9bc60e.png)

Generate new token (classic)をクリックします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/35818eb3-1ead-e0ba-01b7-dc10054a4fbd.png)

repoにチェックを入れます。

`![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/c2c4eb62-c22e-ab55-0728-0b4bec8daf9f.png)

Generate tokenをクリックします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/5a37e8fb-2dbc-de88-73e1-47a6f4fb3776.png)

トークンが表示されたら、忘れずにコピーして安全な場所に保存してください。

### 3. リポジトリのクローン

ポートフォワーディングが確立された状態で、以下のコマンドを使用してリポジトリをクローンします：

```bash
git clone -c "http.proxy=127.0.0.1:8080" http://<username>:<token>/<repository>.git
```

>usernameにはGitHubのユーザー名を、repositoryにはクローンしたいリポジトリ名を入力してください。また、tokenには先ほど生成したトークンを入力してください。

# 注意点

1. **SSHセッションの維持**：このセットアップはSSHセッションがアクティブな間のみ機能します。セッションが切断されると、再度接続を確立する必要があります
1. **セキュリティポリシーの確認**：企業のセキュリティポリシーに違反しないよう、必ず適切な許可を得てから実行してください

# まとめ

SSHローカルフォワーディングを使用することで、直接のインターネットアクセスが制限された環境でも、安全にGitHubリポジトリをクローンすることができます。この方法は、企業のセキュリティ要件を満たしつつ、効率的な開発作業を可能にします。