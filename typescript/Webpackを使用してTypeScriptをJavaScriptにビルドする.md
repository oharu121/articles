# Webpackを使用してTypeScriptをJavaScriptにビルドする

# はじめに

ts-nodeは実行時にTypeScriptをトランスパイルして実行します。これは開発中の迅速なテストやデバッグには便利ですが、トランスパイル処理が毎回実行時に行われるため、実行速度が遅くなる可能性があります。一方、あらかじめTypeScriptをJavaScriptにビルドしておくと、実行時にトランスパイルのオーバーヘッドがなくなるため、アプリケーションの起動時間や実行速度が向上します。

TypescriptのTSCコマンドを使ってjavascriptビルドすることもできますが、複数typescriptを一つのjavascriptにきれいにまとめることはうまくいけませんでした。

# 1. 必要なパッケージのインストール

```
npm install --save-dev webpack webpack-cli ts-loader
```

# 2. Webpackの設定

プロジェクトのルートに `webpack.config.ts` ファイルを作成し、下記の内容を入れます。

```typescript:webpack.config.ts
import * as path from "path";
import { Configuration } from "webpack";

const config: Configuration = {
  mode: "production",
  entry: "./app/main.ts",
  target: "node",
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
    ],
  },
  resolve: {
    extensions: [".tsx", ".ts", ".js"],
    // aliasが設定されている場合
    alias: {
      "@commands": path.resolve(__dirname, "app/commands"),
      "@util": path.resolve(__dirname, "app/util"),
      "@store": path.resolve(__dirname, "app/store"),
      "@class": path.resolve(__dirname, "app/class"),
    },
  },
  node: {
    __dirname: false,
    __filename: false,
  },
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, "dist"),
  },
};

export default config;
```
>* `mode`：ビルドのモードを設定する。
"development"： ちょっとだけ読める
"production"：完全に圧縮され、全然読めない
>* `entry`：ビルドのエントリーポイント。Webpackがバンドル処理を開始するファイルを指定する。
>* `target`：ビルドが対象とする環境。ここでは"node"が指定されており、Node.js環境向けのバンドルが生成される。
>* `module`：モジュールに関する設定。ここでは、rules配列を通じて、異なるファイルタイプに対するローダーの設定を行う。
>* `rules`：ファイルを処理するためのルール（ローダー）を定義する。ここでは、TypeScriptファイル（.tsまたは.tsx）を"ts-loader"を使って処理するように設定している。
>* `resolve`：モジュール解決に関する設定。インポート時の拡張子の省略や、パスのエイリアス設定などを行う。
>* `extensions`：解決するファイルの拡張子を指定する。".tsx"、".ts"、".js"が指定される。
>* `alias`：特定のパスにエイリアスを設定する。これにより、インポート時に相対パスではなくエイリアスを使用できる。
>* `node`：Node.js固有のオプションを設定する。ここでは、__dirnameと__filenameの挙動をカスタマイズしています。
>* `output`：出力に関する設定。バンドルされたファイルの名前や出力先のディレクトリを指定する。
>* `filename`：出力されるバンドルのファイル名。ここでは"bundle.js"が指定されている。
>* `path`：バンドルされたファイルの出力先ディレクトリ。ここでは"dist"ディレクトリが指定されている。

# 3. エントリーポイントの作成

今回の例ではapp配下のmain.tsをエントリーポイントとします。

```typescript:app/main.ts
import RedisServer from "@class/RedisServer";

const server = new RedisServer(process.argv);
server.start();
```

ちなみにインポートしたクラスはこういう感じです。

```typescript:app/class/RedisServer.ts
import { ReplicaConfig } from "types";
import net from "net";
import ping from "@commands/ping";
import { error, info, success, task, warning } from "@util/messageUtil";
import echo from "@commands/echo";
import { set, setWithTimeout } from "@commands/set";
import get from "@commands/get";
import getSectionInfo from "@commands/info";
import { connectToMaster } from "@util/connectUtil";
import { addAck, addReplica, capa } from "@commands/replconf";
import psync from "@commands/psync";
import replicaStore from "@store/replicaStore";
import wait from "@commands/wait";
import * as configStore from "@store/configStore";
import RESPParser from "@class/RESPParser";
import RDBParser from "@class/RDBParser";
import { arrayToRESP } from "@util/IOUtil";
import { getDbFileName, getDbdir } from "@commands/config";
import fs from "fs";
import path from "path";
import rdbStore from "@store/rdbStore";

class RedisServer {
...
}

export default RedisServer;
```

# 4. ビルドスクリプトの追加

`package.json` にビルドスクリプトを追加します。

```diff javascript:package.json
{
  "devDependencies": {
    "@types/node": "^20.11.19",
    "@types/uuid": "^9.0.8",
    "null-loader": "^4.0.1",
    "ts-loader": "^9.5.1",
    "ts-node": "^10.9.2",
    "typescript": "^5.3.3",
    "webpack": "^5.90.2",
    "webpack-cli": "^5.1.4"
  },
  "name": "codecrafters-redis-javascript",
  "description":  "codecrafters-redis-javascript",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "dayjs": "^1.11.10",
    "undici-types": "^5.26.5",
    "uuid": "^9.0.1"
  },
  "scripts": {
    "submit": "npm run build & npm run add & npm run commit & npm run push",
+   "build": "webpack --config webpack.config.ts",
    "add": "git add .",
    "commit": "git commit -m \"comment\"",
    "push": "git push origin master"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}

```
# 5. ビルドの実行

以下のコマンドを実行して、TypeScriptのコードをビルドします。

```
npm run build
```

このコマンドはdist/bundle.jsにビルドされたJavaScriptファイルを生成します。このファイルは、ブラウザで実行可能なJavaScriptコードを含んでいます。

**出力結果**

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/b0cfdba3-7a4a-ea3b-92fa-97f3808193e6.png)

うわああ！全然読めないですね笑

# まとめ

この記事では、ts-nodeを使用したTypeScriptの実行と、Webpackを用いたTypeScriptのビルドプロセスについて説明しました。ts-nodeは開発中の迅速なテストやデバッグに便利ですが、実行時のトランスパイルによりパフォーマンスが低下する可能性があります。一方で、Webpackを使用してTypeScriptをJavaScriptにビルドすることで、実行時のオーバーヘッドを排除し、アプリケーションの起動時間や実行速度を向上させることができます。