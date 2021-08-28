---
title: "GitHub CLI v2.0 のカスタムコマンドを試す"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "githubcli"]
published: true
---

https://cli.github.com/

2021/8/23 に、GitHub CLI 2.0.0 がリリースされました。

```bash
$ gh --version
gh version 2.0.0 (2021-08-23)
https://github.com/cli/cli/releases/tag/v2.0.0
```

新機能には拡張コマンドというものがあり、`gh extension` より拡張コマンドを設定できます。
トピックに、`gh-extension` を付けて公開している人が多いので、好みの拡張コマンドを探すことができそうです。

https://github.com/topics/gh-extension

# 拡張コマンドのインストール

使ってみたい拡張コマンドをインストールしてみます。
`gh extension install ${OWNER}/${REPOSITORY}` コマンドでインストールできます。

## gh-user-status

https://github.com/vilmibm/gh-user-status

GitHub ユーザステータスの確認や変更ができる拡張コマンドです。

```bash
$ gh extension install vilmibm/gh-user-status
Cloning into '/Users/yumenomatayume/.local/share/gh/extensions/gh-user-status'...
remote: Enumerating objects: 133, done.
remote: Counting objects: 100% (133/133), done.
remote: Compressing objects: 100% (96/96), done.
remote: Total 133 (delta 79), reused 85 (delta 37), pack-reused 0
Receiving objects: 100% (133/133), 12.80 MiB | 12.39 MiB/s, done.
Resolving deltas: 100% (79/79), done.
```

`~/.local/share/gh/extensions` に拡張コマンドのリポジトリが clone されているようです。
インストールが完了したら、使用できるかコマンドを実行して確認してみます。

```bash
$ gh user-status get
🏠 Working from home
```

## gh-graph

https://github.com/kawarimidoll/gh-graph

GitHub の Contribution グラフを、ターミナルで表示できます。こちらもインストールしておきます。

```bash
gh extension install kawarimidoll/gh-graph
```

# インストールした拡張コマンドの一覧を出力

`gh extension list` コマンドで、インストールした拡張コマンドの一覧がみれます。

```bash
$ gh extension list
gh graph        kawarimidoll/gh-graph   
gh user-status  vilmibm/gh-user-status
```

# インストールした拡張コマンドのアップグレード

`gh extension upgrade ${OWNER}/${REPOSITORY}` コマンドで、インストールした拡張コマンドをアップグレードできます。
`--all` オプションですべての拡張コマンドをアップグレードします。

```bash
$ gh extension upgrade --all
[graph]: Already up to date.
[user-status]: Already up to date.
```

出力をみる限りだと、`git pull` しているようです。

# インストールした拡張コマンドの削除

`gh extension remove ${OWNER}/${REPOSITORY}` コマンドで、インストールした拡張コマンドを削除できます。

```bash
$ gh extension remove vilmibm/gh-user-status
✓ Removed extension user-status
```

# 拡張コマンドの作成

https://github.com/ymmmtym/gh-gitignore

簡単なものを作成してみました。
GitHub が公開している `.gitignore` のテンプレートを、コマンド実行ディレクトリに保存するだけのカスタムコマンドです。
（今まで新規リポジトリで `.gitignore` を作成するとき、公開しているリポジトリを見に行ってコピペして〜をしていたので、gh コマンドから簡単に作成できるようにしました。）

## 作成手順

以下のディレクトリに移動して作業します。

```bash
cd ~/.local/share/gh/extensions
```

拡張コマンドはこのディレクトリにインストールされるので、ここで作業することにより作成中でもコマンド実行できます。

```bash
$ gh extension create gitignore
✓ Created directory gh-gitignore
✓ Initialized git repository
✓ Set up extension scaffolding

gh-gitignore is ready for development

Install locally with: cd gh-gitignore && gh extension install .

Publish to GitHub with: gh repo create gh-gitignore

For more information on writing extensions:
https://docs.github.com/github-cli/github-cli/creating-github-cli-extensions
```

出力にある通り、作成されたリポジトリに移動します。
元々インストールされるディレクトリで作業しているので、ローカルインストールは不要です。

```bash
cd gh-gitignore
```

`git init` されており、`gh-gitignore` という実行ファイルが存在する状態になっています。

```bash
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
    new file:   gh-gitignore
```

拡張コマンドを実行したとき、実際にはこのファイルが実行されています。
実行ファイルの内容を書き換えていきます。

```bash:gh-gitignore
#!/bin/bash

type fzf >/dev/null 2>&1
if [[ ! $? = 0 ]]; then
  echo -e  "error: fzf command is not found.\n\nPlease install fzf command.\n ex) brew install fzf"
  exit 1
fi

REPO="github/gitignore"
NAME=$(gh api repos/${REPO}/contents --jq .[].name | grep ".gitignore" | fzf)

if [[ ${NAME} = "" ]];then
  echo -e "error: no file selected.\n\nPlease select .gitignore file to download"
  exit 1
fi

curl -s https://raw.githubusercontent.com/${REPO}/master/${NAME} > .gitignore

echo "✓ Downloaded ${NAME} to .gitignore"
```

このカスタムコマンドは、`fzf` が必要なので、インストールされていない場合はエラーが返ってくるようになっています。

ファイル内容を書き換えて GitHub に Push すると、一般的な方法で拡張コマンドをインストールできるようになります。

## 使用方法

拡張コマンドをインストールします。
すでに `~/.local/share/gh/extensions` に配置している場合は不要です。

```bash
gh extension install ymmmtym/gh-gitignore
```

コマンドを実行すると、`github/gitignore` にある `.gitignore` のテンプレートの一覧が表示されるので、取得したいものを選択してダウンロードできます。

```bash
$ gh gitignore 
✓ Downloaded Go.gitignore to /tmp/.gitignore
```

![fzf GitHub gitignore](https://imgur.com/Huipcw0.png)

```bash
$ cat .gitignore 
# Binaries for programs and plugins
*.exe
*.exe~
*.dll
*.so
*.dylib

# Test binary, built with `go test -c`
*.test

# Output of the go coverage tool, specifically when used with LiteIDE
*.out

# Dependency directories (remove the comment below to include it)
# vendor/
```

# さいごに

`gh extension` コマンドのよく使うコマンドと、簡単な拡張コマンドを作成してみました。

コマンドには引数やオプションを指定できるので、汎用的なカスタムコマンドを作成できるように活用します。

# Reference

https://forest.watch.impress.co.jp/docs/news/1346017.html

https://zenn.dev/kawarimidoll/articles/75430b40622e7c

https://github.com/github/gitignore
