## はじめに

在宅勤務やリモート環境で、**VPN経由で会社のActive Directoryドメインにログインする必要がある場合**、新規ユーザーの初回サインイン時に次のようなエラーが表示されることがあります。

```
We cannot sign you in with this credential because your domain isn’t available.
```

![エラー画面](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/e16ea076-a1e0-41a6-a5fa-19ae1103a1a9.png)

## 原因

このエラーは、**Windowsが会社ドメインコントローラに接続できないため、資格情報を確認できない**状況で発生します。

通常、社内LANに直接接続していれば発生しませんが、VPNを使う場合は初回ログイン時に特別な手順が必要です。

**📌 ポイント**

>* 新規ユーザーの資格情報は、まだローカルPCにキャッシュされていません。
>* Windowsはログイン時にドメインコントローラへ問い合わせますが、VPN接続は通常、OSログイン後に確立されるため、この時点では未接続です。
>* そのため「ドメインに接続できない」というエラーになります。

## 1\. ログイン画面で **Network sign-in**

通常のサインインではなく、**ネットワーク経由のサインイン**を選択します。

![Network sign-in](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/ed069761-2d44-4484-a3cf-374eae26103a.png)

## 2\. VPN接続方法を選択

ここで、VPNクライアントの接続オプションが表示されます。
環境によっては「RECOVERY」や「Always On VPN」などの選択肢があります。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/fd2a63b0-c62c-4624-af4e-fc4e7ac2c7ff.png)

## 3\. VPN認証を行う

会社支給のVPNアカウントで認証を行い、VPN接続を確立します。

## 4\. ドメインログインを再試行

VPN接続後に、再びドメインユーザー名とパスワードを入力すると、ドメインコントローラと通信できるためログインが成功します。

## まとめ

>* 新規ユーザーの初回ドメインログインでは、VPN接続が必要
>* ログイン画面の**Network sign-in**からVPNを接続すれば解決
>* 2回目以降はキャッシュされた資格情報でログイン可能
