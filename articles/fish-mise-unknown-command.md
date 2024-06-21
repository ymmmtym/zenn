---
title: "fish 起動時に mise でインストールしたバイナリが Unknown command となるときの対処法"
emoji: "🐟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "fish"]
published: true
---

https://mise.jdx.dev/

mise（ミーズ）とは asdf に変わる CLI 管理ツールです。
最近 asdf から mise に移行しているのですが、fish シェル起動時にエラーが表示されるようになり気になっていました。

## fish 起動時に発生したエラー

`~/.config/fish/config.fish` にインストールしたプラグインを初期化する設定をしているのですが、mise に移行したら以下のようなエラーが出るようになりました。
（出力にはわかりやすく改行を入れています）

```bash
fish: Unknown command: starship
~/.config/fish/config.fish (line 38):
starship init fish | source
^~~~~~~^
from sourcing file ~/.config/fish/config.fish
        called during startup

fish: Unknown command: direnv
~/.config/fish/config.fish (line 43):
direnv hook fish | source
^~~~~^
from sourcing file ~/.config/fish/config.fish
        called during startup

fish: Unknown command: kind
~/.config/fish/config.fish (line 65):
kind completion fish | source
^~~^
from sourcing file ~/.config/fish/config.fish
        called during startup

Welcome to fish, the friendly interactive shell
Type help for instructions on how to use fish
```

エラーになったプラグインとコマンドは以下の通りです。

- starship
  - `starship init fish | source` コマンド
- direnv
  - `direnv hook fish | source` コマンド
- kind
  - `kind completion fish | source` コマンド

ちなみに fish 起動後には正常実行されます。
starship の例ですが、出力の通りプロンプトが切り替わります。

```bash
[I] ~> starship init fish | source

at 03:11:23 🐟 ❯
```

## プラグインが読み込まれない原因

当たり前ですが、fish シェル起動時の`$PATH` 変数に mise のプラグインのパスが記載されていないことが原因でした。

:::details config.fish での $PATH 値
/Users/yumenomatayume/.asdf/shims
/opt/homebrew/opt/asdf/libexec/bin
/opt/homebrew/bin
/opt/homebrew/sbin
/opt/homebrew/opt/mise/bin
/usr/local/bin
/System/Cryptexes/App/usr/bin
/usr/bin
/bin
/usr/sbin
/sbin
/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin
/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin
/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin
:::

:::details fish シェル起動後の $PATH 値
/Users/yumenomatayume/.local/share/aquaproj-aqua/bin
/Users/yumenomatayume/.local/share/mise/installs/kind/latest/bin
/Users/yumenomatayume/.local/share/mise/installs/direnv/latest/bin
/Users/yumenomatayume/.local/share/mise/installs/starship/latest/bin
/Users/yumenomatayume/.local/share/mise/installs/kubebuilder/latest/bin
/Users/yumenomatayume/.asdf/shims
/opt/homebrew/opt/asdf/libexec/bin
/opt/homebrew/bin
/opt/homebrew/sbin
/opt/homebrew/opt/mise/bin
/usr/local/bin
/System/Cryptexes/App/usr/bin
/usr/bin
/bin
/usr/sbin
/sbin
/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin
/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin
/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin
:::

ただこれは mise では想定通りの動きであることが分かりました。
そもそも mise は自身で PATH を書き換えるため、対話型シェル以外では自動的に有効になりません。

https://mise.jdx.dev/dev-tools/shims.html

つまり IDE や起動時に読み込まれるシェルでは mise でインストールしたプラグインが PATH に通っていません。
anyenv や adsf が Shims を採用しているので、移行した際には注意が必要です。

## 対処法

デフォルトの PATH 方式を使い続けるか、Shims 方式に切り替えるかの方法で対処できます。

### 1.  mise のパスを直接読み込む（PATH 方式を使う）

PATH 方式を使い続ける場合、
`~/.config/fish/config.fish` で実行するコマンドを、以下のようなものに変更します。

```bash
mise x -q -- starship init fish | source
```

パスが通ってない時は、`mise x(exec)` コマンドで、インストールしたプラグインを実行できます。

また `mise which` コマンドを実行するとバイナリのパスを取得できます。

```bash
❯ mise which starship
/Users/yumenomatayume/.local/share/mise/installs/starship/latest/bin/starship
```

### 2.  Shims を有効にする（Shims 方式を使う）

Shims 方式を使ってもいい場合、有効化してしまうのが手っ取り早いです。
`~/.config/fish/config.fish` で mise を有効化するときに、`--shims` オプションを追加します。

```bash
mise activate --shims fish | source
```

実際には以下のパス追加が実行されています。これを `~/.config/fish/config.fish` に記載しても同様に Shims が利用できます。

```bash
❯ mise activate --shims
set -gx PATH /Users/yumenomatayume/.local/share/mise/shims $PATH
```

上記の設定をするとプラグインのパスが shims に向きます。

```bash
❯ command -v starship
/Users/yumenomatayume/.local/share/mise/shims/starship
```
