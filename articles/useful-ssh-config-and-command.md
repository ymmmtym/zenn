---
title: "知っておくと便利な SSH の設定やコマンド"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ssh", "linux"]
published: true
---

https://www.openssh.com/

業務で頻繁に SSH を使っていたので、その時に便利だと思ったコマンドや設定を記載します。

## SSH 鍵の生成

- `ssh-keygen` コマンドで SSH 鍵を作成できます。[^1]
  - 秘密鍵（`id_rsa`）と公開鍵（`id_rsa.pub`）がセットで作成されます。

各オプションの説明は以下の通りです。

| オプション | 説明                      | 指定しない場合                                   |
| ----- | ----------------------- | ----------------------------------------- |
| `-t`  | 暗号化方式を指定できます。           | 一般的な RSA 方式の鍵が作成されます。                     |
| `-f`  | 鍵のファイルパスを指定できます。        | 入力を求められます。                                |
| `-b`  | 暗号化強度（鍵の長さ）を指定できます。[^2] | （RSA 方式の場合）`3072bit` になります。 |
| `-N`  | パスフレーズを指定できます。          | 入力を求められます。                                |
| `-C`  | コメントを指定できます。            | 空になります。                                   |

以下のコマンドで、入力を求められずに鍵（`~/.ssh/id_rsa`）を作成できます。

```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ""
```

## ssh-agent

https://nxmnpg.lemoda.net/ja/1/ssh-agent

- 秘密鍵に設定されているパスフレーズを保持できます。
  - 通常パスフレーズが設定されている秘密鍵でログインする時、毎回パスフレーズ入力が必要となります。
  - ssh-agent に秘密鍵を登録しておくことで、パスフレーズ入力が不要になります。
    - 登録時にパスフレーズを入力します。
- ssh-agent に秘密鍵を登録します。

```bash
# ssh-agent起動
$ eval $(ssh-agent)
Agent pid 534

# 鍵の登録
$ ssh-add /path/to/private_key # 鍵のパスを省略するとデフォルトで ~/.ssh/id_rsa になります
Enter passphrase for /root/.ssh/id_rsa: # パスフレーズを入力します。
Identity added: /root/.ssh/id_rsa (root@86db2ef0522a)

# 登録された鍵の確認
$ ssh-add -l # ハッシュ値で表示
3072 SHA256:WqXlrCB8DMPIr7MWzjYJPt8K11FCR2YAa05ThMNBvZM root@86db2ef0522a (RSA)
$ ssh-add -L # 公開鍵で表示
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC8yKniB2GSFBSKhYnDbYmUUhpzPh4VS+hwPkil5DuKpK/rd9GHzQLeYP9+KYg3QynCGTnFCQ/DeBgP56s6ZhyN8Kgw/c7ZLvQNtfYhPtGwgT8N93X/SWxweZ6jDEfQjPmiFxSflYJnYgStsKSs/dtsyAUxQ1n+lkCY3x9Q4PwoqMrAQ/m9zGSXI+bsSt3u0leWbJn53wAmGO7MggS0qHjFctrabRV6CrpAD4VW9LiW79cdS4t24jU4+ojOdeuZYhVlWAhi/RiTeob6ElXRD5sybLa1RLnKBguBJboqdm9ulnjw2r3HbZaEE/n1VGmv2G2Iz3L4mQsL//3+i23QgX/O5wUNV5eCInIDitSAfFTFDrRm/G2j8dPb4MOh4g+IGUJcvOi5EIBcYmeTaVM5ZdADjEbWeWCezOWNUaG+v/NpvGwzU5w2Fu2XNvCMs+RE3s7IAm1GWfPw8YTiYoxH/Kd1Tqmtp+D96jlyCM0zz5DnzDAwdnx0AaqDTGPsB4ZZufM= root@86db2ef0522a
```

- ssh-agent から登録を解除する方法です。[^3]

```bash
# 登録された鍵の削除
$ ssh-add -d /path/to/private_key
Identity removed: /root/.ssh/id_rsa (root@86db2ef0522a)

# ssh-agent停止
$ eval $(ssh-agent -k)
Agent pid 534 killed
```

## SSH コマンドオプション

凡例として、以下の構成になっていることを前提とします。

![https://imgur.com/tQ0AaVe.png](https://imgur.com/tQ0AaVe.png)

### SSH 秘密鍵を指定する

- `-i /path/to/private_key` で指定できます。
- 指定がない場合、`~/.ssh/id_rsa` になります。

```bash
ssh server01 -i ~/.ssh/id_rsa
```

### ログインユーザを指定する

- `$USER@` をログインサーバの頭につける、もしくは `-l $USER` で指定できます。
- 指定がない場合、現在のユーザでログインします。

```bash
ssh ec2-user@server01
ssh server01 -l ec2-user
```

### ログインポートを指定する

- `-p $PORT` で指定できます。

```bash
ssh ec2-user@server01 -p 2222
```

### ポートフォワードする

- `-L <SSH クライアントのポート>:<SSH サーバーのホスト名（IP アドレス）>:<SSH サーバーのポート>`
- リモートにあるサーバーのポートを、ローカルのクライアントにフォワーディングすることで、クライアントのネットワークからリモートのサーバーのポートにアクセスできます。[^4]
  - 左がローカル側、右がリモート側になります。
  - アクセスできない場合は、サーバー側のアクセス権限やファイアフォールを確認しましょう。
- [localhost](http://localhost) の well-known ポートにフォワーディングするときは、root 権限を必要とする場合があります。
  - たとえば、localhost の 443 ポート（未使用）にポートフォワードしたい場合などが該当します。

```bash
# server02の80ポートを、localhostの8080ポートにフォワーディング
ssh server01 -L 8080:server02:80

# localhostに他の端末からのアクセスを待ち受ける時
ssh server01 -L 0.0.0.0:8080:server02:80
ssh server01 -g -L 8080:server02:80

# 別ターミナルで以下を実施
curl -s localhost:8080
```

### エージェントフォワードする

- ssh-agent で登録した秘密鍵の情報を、ログイン先のサーバーへ引継ぎたい時に使用します。
  - 秘密鍵はローカルだけに置いておきたいが、プライベートサブネットに鍵認証して接続したい場合などに用いられます。
- `-A` オプションで ssh-agent-forwarding できます。

```bash
ssh server01 -A

# server01はlocalhostの秘密鍵の情報を引き継いでいます。
ssh-add -l

# この状態で、localhostの秘密鍵を使用して、server02にログインできます。
ssh server02
```

### ログイン先でコマンドを実行する

- ssh コマンドの最後にコマンドを入力することで、ログイン先でコマンドを実行できます。
  - （ログイン状態にならず、）ログイン先でコマンドを実行したい時に使用されます。

```bash
$ ssh server01 hostname
server01
```

### 詳細を確認する

- `-v` オプションで ssh ログインまでのログが表示されます。
- ログインできない時などのトラブルシューティングに有効です。

## ssh_config

https://nxmnpg.lemoda.net/ja/5/ssh_config

https://linux.die.net/man/5/ssh_config

- これまでご紹介したコマンドオプションは、毎回入力するのは手間です。
- `~/.ssh/config` を作成することで、実行ユーザでコマンドオプションが指定されなかったときのデフォルトのオプションを設定できます。（システム全体の設定は `/etc/ssh/ssh_config` に記載します。）
  - 優先される設定情報は以下の順になります。
    - コマンドオプション
    - `~/.ssh/config`
    - `/etc/ssh/ssh_config`
- 設定して便利だったものをご紹介します。
  - 上から順に、`Host` にマッチしたものが読み込まれます。

```bash
Host *                              
  ServerAliveInterval 60
  TCPKeepAlive yes
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  ForwardAgent yes

Host github github.com # GitHubの設定
  HostName github.com
  IdentityFile ~/.ssh/id_rsa.git # GitHubに接続できる鍵を指定
  User git

Host srv01
  HostName server01 # IPアドレスも可能
  IdentityFile ~/.ssh/id_rsa

Host srv02
  Hostname server02
  IdentityFile ~/.ssh/id_rsa
  ProxyCommand ssh -W %h:%p server01
  LocalForward 0.0.0.0:8080:server02:80
  PermitLocalCommand yes
  LocalCommand open https://localhost:8080
```

記載したパラメーターの詳細は以下の通りです。使用頻度が高かったものをまとめました。

| パラメーター                  | 説明                          | 備考                                                |
| ----------------------- | --------------------------- | ------------------------------------------------- |
| `Host`                  | ホスト名（コマンドライン引数で使用するもの）      | ワイルドカード `*` と `?` やスペース区切りで複数指定できます。              |
| `HostName`              | ホスト名 or IP アドレス             | 実際のホスト名か IP アドレスを記載します。                           |
| `IdentityFile`          | 秘密鍵のパス                      | 複数記載すると順番に試されます。`-i` オプションと同じです。                  |
| `User`                  | ログインユーザー                    | `-l` オプションと同じです。                                  |
| `Port`                  | ログインポート                     | `-p` オプションと同じです。                                  |
| `ProxyCommand`          | サーバーに接続するのに使用するコマンド         | サーバー接続前に実行するコマンドです。                               |
| `ForwardAgent`          | フォワーディングエージェント              | `-A` オプションと同じです。                                  |
| `LocalForward`          | ローカルフォワード                   | `-L` オプションと同じです。                                  |
| `TCPKeepAlive`          | サーバーとの接続を保持                 | **yes** or **no** で指定します。                         |
| `ServerAliveInterval`   | サーバー応答がない時にタイムアウトを待つ秒数      | 秒数を指定します。デフォルトは 0 秒です。                            |
| `ServerAliveCountMax`   | `ServerAliveInterval` の実行回数 | 回数を指定します。デフォルトは 3 回です。                            |
| `UserKnownHostsFile`    | `knows_hosts` ファイルパス        | `~/.ssh/knows_hosts` のパスを指定できます。                  |
| `StrictHostKeyChecking` | サーバーのホスト認証鍵の追加              | `~/.ssh/knows_hosts` に接続するサーバーの認証鍵を追加するかどうかの設定です。 |
| `PermitLocalCommand`    | `LocalCommand` の実行を許可       | **yes** or **no** で指定します。                         |
| `LocalCommand`          | サーバーに接続後にローカルで実行するコマンド      | サーバー接続後に実行するコマンドです。                               |

他にも多くのパラメーターがありますので、興味がある方はドキュメントを参照してください。

## さいごに

業務で使っていて便利だったものをまとめてみました。
自分なりの `ssh_config` を作っておくと、サーバーに接続する手間を省くことができるので、参考にしていただけると嬉しいです。

## Reference

参考にしたものです。

### RSA 暗号鍵

https://it-trend.jp/encryption/article/64-0088

https://ssl.sakura.ad.jp/column/decoding-rsa/

### ssh-agent について

https://www.server-memo.net/server-setting/ssh/ssh-agent.html

https://www.fenet.jp/infla/column/server/ssh-agent%E3%81%AE%E4%BD%BF%E3%81%84%E6%96%B9%E3%82%84%E7%A7%98%E5%AF%86%E9%8D%B5%E3%81%AE%E8%BB%A2%E9%80%81%E6%A9%9F%E8%83%BD%E3%82%82%E3%81%94%E7%B4%B9%E4%BB%8B%EF%BC%81/#ssh-agent%E3%81%AE%E4%BE%BF%E5%88%A9%E3%81%AA%E4%BD%BF%E3%81%84%E6%96%B9%E3%82%92%E7%B4%B9%E4%BB%8B%E3%81%97%E3%81%BE%E3%81%99%EF%BC%81

### ポートフォワーディングについて

https://qiita.com/Ayaka14/items/449e2236af4b8c2beb81

http://blue-red.ddo.jp/~ao/wiki/wiki.cgi?page=SSH%A4%CB%A4%E8%A4%EB%A5%DD%A1%BC%A5%C8%A5%D5%A5%A9%A5%EF%A1%BC%A5%C7%A5%A3%A5%F3%A5%B0

https://hrkworks.com/it/windows-tips/openssh/

- -L と -R オプションの違いが図解されており、とてもわかりやすかったです。[^4]

### ssh_config について

https://tech-blog.rakus.co.jp/entry/20210512/ssh

[^1]: OpenSSH のバージョンは `OpenSSH_8.2p1 Ubuntu-4ubuntu0.3, OpenSSL 1.1.1f  31 Mar 2020` です。
[^2]: 現在は、`2048bit` あれば安全と言われています。
[^3]: 実際にはあまり使用しません。ログアウトなど、セッションを終了すると自動的に終了されるためです。
[^4]: リモートホストにポートフォワーディングする `-R` オプションもありますが、実際に使ったことはなかったので割愛します。
