# 長いテキストを自動分割して複数QRコードにするツールを作った話

# はじめに

最近、「大きなテキストデータをQRコードにしたい」というニーズに直面しました。  
しかし、通常のQRコードには文字数制限があり（おおよそ最大1000字程度）、そのままでは入りきりません。  

そこで今回は、**長いテキストを自動で分割し、複数のSVG形式のQRコードとして出力する簡単なツール**を作ったので紹介します。

使用した主な技術は以下です。

- Node.js
- `fs`モジュール（ファイル読み書き）
- `qrcode`パッケージ（QRコード生成）

https://www.npmjs.com/package/qrcode

# 処理の流れを解説

1. **入力ファイルの読み込み**
   - 指定された`path`のテキストファイルを読み込みます。
2. **文字数制限ごとに分割**
   - 1チャンク最大1000文字ずつに分割します。
3. **HTMLファイルの初期化**
   - 出力先`result.html`を空にして、HTMLのヘッダー部分を書き込みます。
4. **各チャンクをQRコードに変換して出力**
   - 各チャンクをSVG形式のQRコードに変換し、順番にHTMLファイルに追記していきます。
5. **HTMLの締め**
   - 最後に`</body>`と`</html>`を追加してHTMLを完成させます。
6. **生成されたファイルを自動で開く**
   - `this.openFile(outputDir)`で、生成されたQRコード一覧HTMLをすぐにブラウザなどで確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/dea1cd4a-55c6-49da-bb64-7efdedb29775.png)

# 実装コード

作成したメインメソッドは以下の通りです。

```typescript:FileUtil.ts
public genQrcode(path: string): void {
  const MAX_QR_CHARS = 1000;
  const chunks: string[] = [];
  const outputDir = "output/qr_parts/result.html";
  const input = fs.readFileSync(path, "utf8").replace(/:\/\//g, "");

  for (let i = 0; i < input.length; i += MAX_QR_CHARS) {
    chunks.push(input.slice(i, i + MAX_QR_CHARS));
  }

  fs.writeFileSync(outputDir, "");
  fs.appendFileSync(
    outputDir,
    [
      `<!DOCTYPE html>`,
      `<html>`,
      `<head>`,
      `<title>SVG parts</title>`,
      `</head>`,
      `<body>\n`,
    ].join("\n")
  );

  chunks.forEach((chunk, i) => {
    QRCode.toString(chunk, { type: "svg" }, function (_err, url) {
      fs.appendFileSync(outputDir, `<h1>part ${i + 1}</h1>\n`);
      fs.appendFileSync(outputDir, url);
    });
  });

  fs.appendFileSync(outputDir, [`</body>`, `</html>`].join("\n"));
  this.openFile(outputDir);
}
```

**🔨 使い方**

>1. 長いテキストファイルを用意する（例: `input.txt`）
>2. コードの`path`にファイルパスを指定して`genQrcode`を呼び出す
>3. `output/qr_parts/result.html`が生成され、ブラウザで開くとチャンクごとのQRコード一覧が見れる！

**👷 工夫したポイント**

>- **ファイルを一気に作るのではなく、チャンクごとに少しずつ追記する**ことで、大量データにも対応できるようにしています。
>- **SVG形式で出力**しているので、解像度を気にせずきれいなQRコードが表示できます。
>- それぞれのチャンクに「part n」という見出しをつけて、どの部分かわかりやすくしています。

**⚠️ 注意点**

>- 1000文字という制限は、データの内容（日本語/英語や記号の比率）によってはもっと厳しくなる場合があります。
>- ファイルサイズが大きすぎる場合、ブラウザで開くのに少し時間がかかるかもしれません。
>- インデントや空白が全部なくなり、1行でベタっと出力されてしまいます。
>- テキストに`://`全体がURLエンコードされてしまうので、`replace(/:\/\//g, "")`で退避

# まとめ

長文を複数QRコードに分割するニーズは意外と多くあります。  
今回のツールは、 **「とりあえず分割して一覧化したい！」** という用途にぴったりです。

今後、さらに改善するなら、
- ダウンロードボタンを付ける
- ZIP圧縮でまとめて保存
- PDFとしてまとめる
などにもチャレンジしていきたいです！
