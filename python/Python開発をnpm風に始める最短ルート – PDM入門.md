# Python開発をnpm風に始める最短ルート – PDM入門

# はじめに

Python開発の依存管理に「Poetry」や「pipenv」を使ったことがある人も多いはず。

しかし、Node.js の `npm` / `yarn` に慣れている人からすると、Poetry はカチッとしすぎるし、pipenv は普及力にかける...

そんな中、最近人気が出ているのが `PDM (Python Development Master)` です。

* `npm install` のような簡単なコマンド
* `npm run` のようなタスク定義
* `pyproject.toml` だけで完結
* モジュール化も自由

この記事では「PDM とは何か」から、「これぞ、Python の npm だと思った理由」まで簡潔に紹介します。

# PDM とは？

PDM (Python Development Master) は、PEP 582 / pyproject.toml 基盤の現代的パッケージ管理ツールで、

* Poetry ほどカチッとしておらず
* pipenv より深くモダン

な「続けられるツール」として人気を集めています。

https://pdm.fming.dev/latest/

**🌐 Node.js との対応関係**

| 機能                | npm / yarn             | PDM                   |
| ----------------- | ---------------------- | --------------------- |
| init              | `npm init`             | `pdm init`            |
| install package   | `npm install axios`    | `pdm add requests`    |
| uninstall package | `npm uninstall axios`  | `pdm remove requests` |
| run script        | `npm run start`        | `pdm run start`       |
| lock file         | `package-lock.json`    | `pdm.lock.toml`       |
| 使用環境              | global / node\_modules | ローカル環境 or PEP 582     |

#  PDM のセットアップ

**Windows**

```bash
powershell -ExecutionPolicy ByPass -c "irm https://pdm-project.org/install-pdm.py | py -"
```

**Mac**

```bash
curl -sSL https://pdm-project.org/install-pdm.py | python3 -
```

# プロジェクトのセットアップ

### 1. プロジェクト初期化

```bash
mkdir my-app && cd my-app
pdm init -n
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/15a60b07-5538-492f-87db-5737bc502b64.png)

### 2. 依存関係の追加

```bash
pdm add requests
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3760374/d75415af-3b98-4017-ab68-cf847d28077d.png)

### 3. スクリプト実行

```py:main.py
import requests

def main():
    res = requests.get("https://httpbin.org/get")
    print(res.status_code)

if __name__ == "__main__":
    main()
```

```bash
# 実行
pdm run python main.py
```

# npm run 風にコマンド定義

```toml:pyproject.toml
# 下記を追加
[tool.pdm.scripts]
start = "python main.py"
```

```bash
# 実行
pdm run start
```

# まとめ

* やっと出てきた「本当の npm 風エンジニア向け」Pythonツール
* ライブラリよりアプリ開発者向け
* 試してみる価値あり

**🌟 PDM の良いところ**

* `npm run` 感覚の task runner
* パッケージにしなくてもOK
* Poetryよりライト
* pip / venv / pipfile からの移行も楽
* 開発者のアプリに最適

**🚫 PDM が合わない場合**

* 現時点では CI/CD のテンプレートが少なめ
* まだ少数派（Poetryの方が社会的デファクト）
* PyPI への公開ライブラリ開発なら Poetry が精密

補足：Poetryのご紹介はこちら：

https://qiita.com/oharu121/items/bb2ad2a89549db373af5
