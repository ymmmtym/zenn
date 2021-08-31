---
title: "VS Code の拡張機能 GistPad を使ってみる"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "github", "githubgist"]
published: true
---

https://marketplace.visualstudio.com/items?itemName=vsls-contrib.gistfs

VS Code で GitHub Gist と連携できる「GistPad」という拡張機能があったので使ってみました。
5 分もあれば VS Code から Gist を操作が可能になります。

## 拡張機能のインストール

拡張機能のメニューから、「gistpad」で検索すると見つかります。こちらをインストールします。

![GistPad Install](https://imgur.com/zlDevLo.png)

インストールが完了すると、ノートみたいな GistPad のアイコンが追加されます。

## GitHub にログイン

左のメニューバーより、**Sign In**をクリックするとアクセス権を許可するかどうかの画面に遷移します。

![GistPad Login](https://imgur.com/WKGNPIT.png)

Continue より認証を進めていきます。

![GistPad Auth](https://imgur.com/IfNQ6kR.png)

アクセスを許可すると、左のメニューバーに Gist 一覧が表示されます。
個人の Gist と、Star したものを表示してくれます。

![GistPad Editor](https://imgur.com/GZaLDeo.png)

**New scratch note...** より、新しい Gist を作成できます。
（実際に `test` と言う名前の Gist に、`test.md` を作成しました。）

![GistPad Sample](https://imgur.com/BrMiG6r.png)

Gist 側にもすぐに反映されているので、本当に、VS Code だけで Gist を操作している感じです。

Gist に上げておくと、Zenn にも簡単に埋め込みができます。

@[gist](https://gist.github.com/ymmmtym/52962507f3e5b43e86fc08d08a1edb52?file=test.md)

## その他の拡張機能

他にも Gist を操作する拡張機能があります。[^1]

どちらも PAT(Personal Access Token) が必要なので、事前に作成しておきます。

### Gist

https://marketplace.visualstudio.com/items?itemName=kenhowardpdx.vscode-gist

Gist のファイルをローカルに落として、編集した内容を同期する処理がされているようです。VS Code だけでなく、他のエディターを使用したい場合は上記で使用できます。

### GitHub Gist Explorer

https://marketplace.visualstudio.com/items?itemName=k9982874.github-gist-explorer

GistPad とほぼ同じです。GistPod の方が多くの機能があるように感じました。

## さいごに

GistPad は、Gist だけでなく GitHub リポジトリのファイル修正もできます。（更新したタイミングでコミットされるので注意です。）

他にもさまざまなことができるので、公式サイトをみながら活用してみます。

## Reference

参考にしたものです。

https://qiita.com/kai_kou/items/ceeee47996339e5eecc4

[^1]: はじめは、Gist という拡張機能を使うつもりでしたが、実際に使ってみると VS Code 上でファイルがブラウズできないなど不便な点があったので、GistPad に乗り換えました。
