---
title: "GitHub CLI v2.0 のカスタムコマンドを試す"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "githubcli"]
published: false
---

https://cli.github.com/

2021/8/23 に、GitHub CLI 2.0.0 のメジャーバージョンがリリースされました。

```bash
$ gh --version
gh version 2.0.0 (2021-08-23)
https://github.com/cli/cli/releases/tag/v2.0.0
```

新機能には拡張コマンドというものがあり、`gh extension` よりカスタムコマンドを実行できます。
トピックに、`gh-extension` を付けて公開している人が多いので、好みの拡張機能を探すことができそうです。

https://github.com/topics/gh-extension

# インストールしてみる

使ってみたい拡張機能をインストールしてみます。

## gh-user-status

https://github.com/vilmibm/gh-user-status

GitHub のユーザ状態の確認や変更ができる拡張機能です。
`gh extension install` コマンドでインストールできます。

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

`~/.local/share/gh/extensions` に拡張機能のリポジトリが clone されているようです。
インストールが完了したら、コマンドを実行してみます。

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

## インストールした一覧の確認

`gh extension list` コマンドで、インストールした拡張機能の一覧がみれます。

```bash
$ gh extension list
gh graph        kawarimidoll/gh-graph   
gh user-status  vilmibm/gh-user-status
```

# アップグレードしてみる

`gh extension upgrade` コマンドで、インストールした拡張機能をアップグレードできます。

出力をみる限りだと、`git pull` しているようです。

```bash
$ gh extension upgrade --all
[graph]: Already up to date.
[user-status]: Already up to date.
```

# 削除してみる

`gh extension remove` コマンドで、インストールした拡張機能を削除できます。

```bash
$ gh extension remove vilmibm/gh-user-status
✓ Removed extension user-status
```

# 自分で作ってみる

https://github.com/ymmmtym/gh-gitignore

簡単なものを作成してみました。GitHub が公開している `.gitignore` のテンプレートを取得するだけのカスタムコマンドです。
（今まで新規リポジトリで `.gitignore` を作成するとき、公開しているリポジトリを見に行ってコピペして〜をしていたので、gh コマンドから簡単に作成できるようにしました。）

## 作成手順

以下のディレクトリに移動して作業します。

```bash
cd ~/.local/share/gh/extensions
```

拡張機能はこのディレクトリにインストールされるので、ここで作成すると GitHub に上げなくても使うことができます。
また、作成時の出力にある通りローカルインストールもできますが元々インストールされるディレクトリで作業しているので、インストールは不要です。

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

作成されたリポジトリに移動します。

```bash
cd gh-gitignore
```

`git init` されており、`gh-gitignore` という実行ファイルが存在する状態になっています。
このファイルに実行内容を記載します。

```bash
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
実行内容の記載が完了して GitHub に Push すると、すぐに使えるようになりました。

```bash
gh extension install ymmmtym/gh-gitignore
```

任意のディレクトリで実行できます。

```bash
$ gh gitignore 
✓ Downloaded Go.gitignore to .gitignore

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

`gh extension` コマンドのよく使うコマンドと、簡単なカスタムコマンドを作成してみました。

コマンドには引数やオプションを指定することができるので、汎用的なカスタムコマンドを作成できるように活用したいと思います。

# Reference

https://forest.watch.impress.co.jp/docs/news/1346017.html

https://zenn.dev/kawarimidoll/articles/75430b40622e7c
