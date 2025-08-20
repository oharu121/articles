## anyConnect のログインを自動化させる

## はじめに

在宅勤務をしている人の多くは VPN 接続が必要だと思います。私にとって、毎回パソコンを起動するたびにログインして VPN に接続するのは本当に面倒です。今日は Cisco の AnyConnect を例に、バッチを使用して自動化を実現する方法を見てみましょう。

**⭐ 今日の主役：vpncli.exe**

vpncli.exe は、コマンドラインインターフェースを通じて VPN 接続を管理するための実行可能ファイルです。特に Cisco の AnyConnect VPN クライアントにおいて、この実行ファイルを使用して接続、切断、ステータス確認などの操作をコマンドラインから実行できます。これにより、GUI を使用せずに VPN 接続の自動化やスクリプト化が可能になります。

今日は vpncli.exe を使って VPN 接続を自動化したいと思います。

## 1. バッチの実装

下記のようにバッチを作成します。

```batch:autoAnyConnect.bat
@echo off
taskkill -im csc_ui.exe -f
cd "C:\Program Files (x86)\Cisco\Cisco Secure Client\"
echo Connecting to VPN...
vpncli.exe -s < script.txt
```

**📌 ポイント**

> - vpncli.exe の実際の保存場所は異なる場合がありますので、実際の状況に応じて調整してください。

## 2. 起動スクリプトの実装

下記のように script.txt を作成し、「C:\Program Files (x86)\Cisco\Cisco Secure Client\」の下に張り付けます。

```text:script.txt
connect <接続先のドメイン名またはIPアドレス>
<AnyConnectのユーザ名>
<AnyConnectのパスワード>
```

**📌 ポイント**

> - 貼り付け先は vpncli.exe の保存場所に合わせてください。

これで autoAnyConnect.bat をクリックするだけで VPN 接続を実施してくれます！

## 補足：起動時実施するためには

Windows キー）と「R」キーを同時に押すと、「ファイル名を指定して実行」ウィンドウが開きます。「shell:startup」を入力し OK ボタンを押すと Startup フォルダが開けます。
![startup.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/0a0d0753-517a-cd5e-37b5-02408ccae08a.png)
Startup フォルダに autoAnyConnect.bat を貼り付けると毎回起動時実行してくれます。

## 終わりに

この記事では、在宅勤務で VPN 接続が必要な方々向けに、Cisco の AnyConnect VPN クライアントを使用した接続プロセスの自動化方法を紹介しました。コマンドラインツール vpncli.exe を用いたバッチファイルの作成から、スタートアップに追加することで PC 起動時に自動接続する設定まで、手順を詳しく解説しています。この自動化により、VPN 接続の手間を省き、よりスムーズで効率的な在宅勤務環境を実現できます。
