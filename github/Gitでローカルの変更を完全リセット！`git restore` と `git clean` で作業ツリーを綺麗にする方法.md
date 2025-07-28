# Gitでローカルの変更を完全リセット！`git restore` と `git clean` で作業ツリーを綺麗にする方法

# はじめに

「間違えてファイルを編集してしまった…」
「ビルド成果物や不要なログファイルが散らかってしまった…」

Gitでの作業中、ローカルの作業ディレクトリを**最後のコミット直後のきれいな状態**に完全に戻したくなることがあります。

そんなときに役立つ、2つの強力なコマンドを解説します。

>* `git restore .`：**追跡中ファイル**の編集内容を取り消す
>* `git clean -fd`：**未追跡ファイル**を完全に削除する

⚠️ **注意：** これから紹介するコマンドは、ローカルの変更を恒久的に削除します。元に戻すことはできないため、慎重に使用してください。


# こんな状況、ありませんか？

コーディングやビルドの途中で `git status` を実行すると、よくこんな状態になっています。

```bash
$ git status

On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   src/components/Header.js  # 編集してしまったファイル

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        dist/                               # ビルドで生成されたディレクトリ
        temp.log                            # 一時的なログファイル

no changes added to commit (use "git add" and/or "git commit -a")
```

このシナリオでは、以下の2つの問題を解決します。

>1.  **編集してしまった** `src/components/Header.js` を元に戻したい。
>2.  **新しく作られた** `dist/` ディレクトリと `temp.log` ファイルを削除したい。

# 1. 編集内容を取り消す (`git restore`)

Gitが追跡している（一度でもコミットしたことがある）ファイルの変更を元に戻すには、`git restore` コマンドを使います。

```bash
git restore .
```

これを実行すると、**ステージされていないすべての変更（`modified`状態のファイル）が、最後のコミット（HEAD）の状態まで巻き戻されます。**

ドット（`.`）は「カレントディレクトリ以下のすべてのファイル」を意味します。

> **💡 モダンな方法を使おう**
以前は `git checkout .` がこの目的で使われていましたが、`checkout` はブランチ切り替えなど多機能で紛らわしいものでした。Git 2.23から導入された `git restore` は、**作業ツリーの変更を取り消す**という役割が明確なため、こちらの使用が強く推奨されます。

# 2. 未追跡ファイルを一掃する (`git clean`)

次に、Gitの管理下にない新しいファイル（Untracked files）を削除します。これには `git clean` コマンドが最適です。

```bash
git clean -fd
```

このコマンドは、Gitに追跡されていないファイルやディレクトリを一括で削除します。オプションの意味は以下の通りです。

| オプション | 意味 | 説明 |
| :--- | :--- | :--- |
| `-f` | **f**orce | 「ファイルを本当に削除します」という意思表示。安全のため必須のオプションです。 |
| `-d` | **d**irectory | ファイルだけでなく、ディレクトリも削除対象に含めます。`dist/`のようなフォルダを消すのに必要です。|

**⚠️ 最重要：実行前に必ず `--dry-run` で確認！**

`git clean` は強力なコマンドなので、誤って必要なファイルを消してしまわないように、まずは **`--dry-run`** オプションをつけて「何が削除されるか」だけを確認する癖をつけましょう。

```bash
# 削除はせずに、対象となるファイル・ディレクトリの一覧を表示する
$ git clean -fd --dry-run

Would remove dist/
Would remove temp.log
```

>`Would remove ...` のリストを見て、消しても問題ないことを確認してから、`--dry-run` を外して本番のコマンドを実行しましょう。

# 合わせ技：作業ディレクトリを完全にリセットする

これら2つのコマンドを組み合わせれば、作業ディレクトリを「まっさら」な状態に戻せます。

```bash
# ステップ1: 追跡中ファイルの変更を元に戻す
git restore .

# ステップ2: 未追跡ファイルを削除する
git clean -fd
```

これで、`git status` を実行しても `nothing to commit, working tree clean` と表示される、クリーンな状態が手に入ります。

# まとめ

| やりたいこと | 推奨コマンド |
| :--- | :--- |
| **編集したファイル**を元に戻したい | `git restore .` |
| **未追跡ファイル**を削除したい | `git clean -fd` |
| 削除されるファイルを**事前に確認**したい | `git clean -fd --dry-run` |
| **すべてのローカル変更をリセット**したい | `git restore . && git clean -fd` |

急いでいるときほど、これらのコマンドは頼りになります。ただし、その破壊力を忘れずに、特に `git clean` を使う前には必ず `--dry-run` で安全確認を行いましょう。