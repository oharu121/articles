## 社内で SSH を使用して GitHub リポジトリをクローンする方法

## はじめに

企業環境ではセキュリティの観点から、社内ネットワークからのインターネットアクセスが制限されていることがよくあります。

こうした状況でも、リモートサーバを経由して GitHub リポジトリをクローンする方法として、「SSH ローカルフォワーディング」が有効です。

この記事では、ローカルフォワーディングを活用して GitHub に安全にアクセスする手順を詳しく解説します。

**前提条件**

> 1.  ローカルマシンに SSH クライアントがインストールされていること
> 1.  リモートサーバへの SSH アクセス権限があること
> 1.  リモートサーバが GitHub にアクセスできること
> 1.  GitHub アカウントとアクセスしたいリポジトリへの権限があること

## 1. SSH ローカルフォワーディングの設定

まず、ローカルマシンからリモートサーバへの SSH 接続を確立し、ポートフォワーディングを設定します。

```bash
ssh <username>@<remote-server> -L 8080:localhost:80
```

> username はリモートサーバのユーザー名に、remote-server はリモートサーバのアドレスに置き換えてください。

## 2. GitHub でのトークン作成

右上のプロフィールアイコンをクリックし、Settings を選択します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/90687b76-953a-b261-097b-8115f297c84c.png)

左サイドメニューから Developer settings をクリックします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/b4ed5ff1-42b7-2508-eb3b-e1e97feb823c.png)

Personal access tokens > Tokens (classic)を選択します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/07414576-c758-bd6c-f6ba-851c8e9bc60e.png)

Generate new token (classic)をクリックします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/35818eb3-1ead-e0ba-01b7-dc10054a4fbd.png)

repo にチェックを入れます。

`![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/c2c4eb62-c22e-ab55-0728-0b4bec8daf9f.png)

Generate token をクリックします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/5a37e8fb-2dbc-de88-73e1-47a6f4fb3776.png)

トークンが表示されたら、忘れずにコピーして安全な場所に保存してください。

#### 3. リポジトリのクローン

ポートフォワーディングが確立された状態で、以下のコマンドを使用してリポジトリをクローンします：

```bash
git clone -c "http.proxy=127.0.0.1:8080" http://<username>:<token>/<repository>.git
```

> username には GitHub のユーザー名を、repository にはクローンしたいリポジトリ名を入力してください。また、token には先ほど生成したトークンを入力してください。

## 注意点

1. **SSH セッションの維持**：このセットアップは SSH セッションがアクティブな間のみ機能します。セッションが切断されると、再度接続を確立する必要があります
1. **セキュリティポリシーの確認**：企業のセキュリティポリシーに違反しないよう、必ず適切な許可を得てから実行してください

## まとめ

SSH ローカルフォワーディングを使用することで、直接のインターネットアクセスが制限された環境でも、安全に GitHub リポジトリをクローンすることができます。この方法は、企業のセキュリティ要件を満たしつつ、効率的な開発作業を可能にします。
