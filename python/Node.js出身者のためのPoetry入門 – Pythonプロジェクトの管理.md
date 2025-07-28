# Node.js出身者のためのPoetry入門 – Pythonプロジェクトの管理

# はじめに

Node.js では `npm init` を叩けば `package.json` が作られて依存管理の準備が整いますよね？

Pythonにも似たような仕組みがありますが、標準の `pip` + `venv` ではやや物足りない…

そこで今回は，**Pythonの依存管理ツール「Poetry」** を使って、Node.jsとの比較を交えながらプロジェクトセットアップの6段階を簡潔に紹介します。

# Poetryとnpmの対応関係

| 概念        | Node.js (`npm`)     | Python (`poetry`)           |
| --------- | ------------------- | --------------------------- |
| 初期化コマンド   | `npm init`          | `poetry init`               |
| 依存追加      | `npm install axios` | `poetry add requests`       |
| 依存リストファイル | `package.json`      | `pyproject.toml`            |
| ロックファイル   | `package-lock.json` | `poetry.lock`               |
| 実行環境の隔離   | `node_modules`      | 仮想環境 (`.venv`)              |
| 実行        | `node script.js`    | `poetry run python main.py` |

# Poetryのセットアップ手順

### 1. Pythonの確認

先に使用可能なPythonがあることを確認：

```bash
python --version
```

```bash
# 出力
Python 3.13.1
```

### 2. Poetry のインストール

```bash
curl -sSL https://install.python-poetry.org | python -
```

### 3. PATHに追加

Poetryは Windows では以下のパスにインストールされます：

```
C:\Users\<UserName>\AppData\Roaming\Python\Scripts
```

これを簡単に追加するPowerShellコマンド：

```powershell
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";C:\Users\<ユーザ名>\AppData\Roaming\Python\Scripts", "User")
```

ターミナルを閉じて再起動し、以下を実行：

```bash
poetry --version
```

### 4. 仮想環境をプロジェクト内に作る設定

> ※一度だけでOK

```bash
poetry config virtualenvs.in-project true
```

これを実行すると、今後作成するすべてのプロジェクトで `.venv` フォルダがプロジェクト直下に作られます。

* Node.js の `node_modules/` のように、**環境が見える形で管理できる**
* VSCode でも自動的に Python interpreter として認識される
* 一度設定すれば永続的に有効

# プロジェクトのセットアップ

### 1. プロジェクト初期化

```bash
mkdir my-poetry-app
cd my-poetry-app
poetry init
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/aa9be309-7321-4b16-b5e1-da64c9c7b3b0.png)

問答形式で設定するか，一気に指定したければ以下も可能：

```bash
poetry init --name "my-poetry-app" --dependency requests
```

### 2. 依存関係の追加

```bash
poetry add requests
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/28b48329-f252-400b-b2f2-a3dec09a4657.png)


### 3. スクリプト実行
```py:main.py
import requests

response = requests.get("https://httpbin.org/get")
print(response.status_code)
print(response.json())
```

```bash
# 実行
poetry run python main.py
```

### 4. 他人の環境で再現するには
```bash
git clone ...
cd your-project
poetry install
```

これだけで .venv と依存が自動構築されます。

# まとめ

Node.jsの開発体験に慣れている人にとって、Poetryはごく自然に入っていける設計になっています。

| 作業内容      | npm コマンド        | poetry コマンド             |
| --------- | --------------- | ----------------------- |
| プロジェクト初期化 | `npm init`      | `poetry init`           |
| 依存追加      | `npm install`   | `poetry add`            |
| 実行        | `node index.js` | `poetry run python ...` |
| 再現        | `npm install`   | `poetry install`        |

Pythonの世界でも、**安心してモダンな依存管理**を始めていきましょう！

✨ よりモダンで自由なnpm風パッケージ管理ツールのご紹介はこちら：

https://qiita.com/oharu121/items/d09c333bb30e90e2db64
