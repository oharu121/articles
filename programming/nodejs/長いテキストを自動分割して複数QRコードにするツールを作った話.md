## 長いテキストを自動分割して複数 QR コードにするツールを作った話

## はじめに

最近、「大きなテキストデータを QR コードにしたい」というニーズに直面しました。  
しかし、通常の QR コードには文字数制限があり（おおよそ最大 1000 字程度）、そのままでは入りきりません。

そこで今回は、**長いテキストを自動で分割し、複数の SVG 形式の QR コードとして出力する簡単なツール**を作ったので紹介します。

使用した主な技術は以下です。

- Node.js
- `fs`モジュール（ファイル読み書き）
- `qrcode`パッケージ（QR コード生成）

https://www.npmjs.com/package/qrcode

## 処理の流れを解説

1. **入力ファイルの読み込み**
   - 指定された`path`のテキストファイルを読み込みます。
2. **文字数制限ごとに分割**
   - 1 チャンク最大 1000 文字ずつに分割します。
3. **HTML ファイルの初期化**
   - 出力先`result.html`を空にして、HTML のヘッダー部分を書き込みます。
4. **各チャンクを QR コードに変換して出力**
   - 各チャンクを SVG 形式の QR コードに変換し、順番に HTML ファイルに追記していきます。
5. **HTML の締め**
   - 最後に`</body>`と`</html>`を追加して HTML を完成させます。
6. **生成されたファイルを自動で開く**
   - `this.openFile(outputDir)`で、生成された QR コード一覧 HTML をすぐにブラウザなどで確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/dea1cd4a-55c6-49da-bb64-7efdedb29775.png)

## 実装コード

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

> 1.  長いテキストファイルを用意する（例: `input.txt`）
> 2.  コードの`path`にファイルパスを指定して`genQrcode`を呼び出す
> 3.  `output/qr_parts/result.html`が生成され、ブラウザで開くとチャンクごとの QR コード一覧が見れる！

**👷 工夫したポイント**

> - **ファイルを一気に作るのではなく、チャンクごとに少しずつ追記する**ことで、大量データにも対応できるようにしています。
> - **SVG 形式で出力**しているので、解像度を気にせずきれいな QR コードが表示できます。
> - それぞれのチャンクに「part n」という見出しをつけて、どの部分かわかりやすくしています。

**⚠️ 注意点**

> - 1000 文字という制限は、データの内容（日本語/英語や記号の比率）によってはもっと厳しくなる場合があります。
> - ファイルサイズが大きすぎる場合、ブラウザで開くのに少し時間がかかるかもしれません。
> - インデントや空白が全部なくなり、1 行でベタっと出力されてしまいます。
> - テキストに`://`全体が URL エンコードされてしまうので、`replace(/:\/\//g, "")`で退避

## まとめ

長文を複数 QR コードに分割するニーズは意外と多くあります。  
今回のツールは、 **「とりあえず分割して一覧化したい！」** という用途にぴったりです。

今後、さらに改善するなら、

- ダウンロードボタンを付ける
- ZIP 圧縮でまとめて保存
- PDF としてまとめる
  などにもチャレンジしていきたいです！
