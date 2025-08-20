## はじめに
npm や `npx playwright install` などのコマンド実行時に、以下のようなエラーを見たことはありませんか？

```
Error: self-signed certificate in certificate chain
```

これは **SSL 証明書の検証に失敗した** ことを意味します。  
特に以下のような環境で発生しやすいです。

>- 社内プロキシ経由でインターネットにアクセスしている
>- 社内独自の認証局 (CA) を使用している
>- 自己署名証明書（self-signed certificate）を使用している

この記事では、このエラーの仕組みと、**安全な方法** と **一時的な回避方法** の両方を解説します。

## エラーの仕組み

`npm` や `npx` は、依存パッケージやブラウザバイナリを HTTPS 経由でダウンロードします。  通常は公式の証明書で検証されますが、企業ネットワークでは以下の理由で失敗する場合があります。

>- プロキシが HTTPS 通信を復号して再暗号化している（中間者証明書を使用）
>- サーバー証明書が自己署名されている
>- 中間証明書が信頼されていない

その結果、「証明書チェーンに自己署名証明書が含まれている」というエラーが発生します。

## 1\. **npm の設定で回避**

SSL 検証をオフにします（危険なので開発環境のみ推奨）。

```bash
npm set strict-ssl false
npm set registry http://registry.npmjs.org/
```

## 2\. **Node.js の証明書検証を無効化**

一時的に TLS 検証を完全に無効化します。

**Windows（cmd）:**

```cmd
set NODE_TLS_REJECT_UNAUTHORIZED=0
```

**Windows（PowerShell）:**

```powershell
$env:NODE_TLS_REJECT_UNAUTHORIZED=0
```

## 3\. 実施後設定を戻す

作業が終わったら元に戻します。

```bash
npm set strict-ssl true
$env:NODE_TLS_REJECT_UNAUTHORIZED=1
```

## まとめ

>- 本番環境や日常的な開発環境では **証明書を正しく信頼させる方法（`NODE_EXTRA_CA_CERTS`）** が安全
>- 急ぎの検証や一時的回避なら **`strict-ssl false` や `NODE_TLS_REJECT_UNAUTHORIZED=0`**
>- いずれも作業後は元の設定に戻し、セキュリティリスクを最小限にする
