---
title: "inadyn+docker で自宅 DNSドメインを自動更新する"
emoji: "💽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dns", "inadyn", "docker", "kubernetes"]
published: true
---

DNS ドメインの取得に、Google Domains を利用しています。
Google Domains はダイナミック DNS（DDNS）が提供されており、動的に変わる DNS レコードも登録できます。

https://domains.google/intl/ja_jp/?gclid=Cj0KCQiAuP-OBhDqARIsAD4XHpeUtYL4y6Gw-N9_PXgQNfy_Rg8bUEO7NqjYefMWxbcxQ_vzqnu4Jv8aAmFyEALw_wcB&gclsrc=aw.ds

以前までは、`お名前.com` と no-ip（無料 DDNS プロバイダー）を利用しており、以下のような運用をしていました。

- `お名前.com` でドメインを取得（購入）する
  - 取得したドメインを、クラウドや SaaS などのカスタムドメインに設定する
- no-ip で無料 DDNS ドメインを取得する
  - 自宅ルーターを no-ip と連携させて、自動更新されるようにする
  - `お名前.com` で設定したカスタムドメインを、自宅用のドメインを no-ip で取得したドメインにリダイレクト（CNAME を設定）する

https://blog.yumenomatayume.net/2021/08/15/01

Google Domains に移行してから、すべてのレコードが一元的に管理できるようになりました。その時に導入した inadyn について記載します。

## inadyn とは

https://github.com/troglobit/inadyn

https://www.freebsd.org/cgi/man.cgi?query=inadyn&apropos=0&sektion=8&manpath=FreeBSD+11-current&format=html

inadyn[^1] は OSS の DDNS Client です。Mac / Linux で利用でき、Docker イメージも配布されています。さらに、arm / amd アーキテクチャーに対応しているので、ラズパイでも動かすことができます。

Google Domains をはじめ、サポートしている DDNS プロバイダーが豊富にあります。（詳しくは GitHub の README に記載されています。）

## inadyn を使ってみる

登録する DNS ドメインは以下になります。

- ドメイン名: `home.yumenomatayume.net`
- IP アドレス: `219.100.37.238` [^2]

### 認証情報の取得

DDNS Client を使用してドメインを更新するためには、DDNS プロバイダーの認証が必要です。

Google Domains の場合は、「認証情報を表示」をクリックしてユーザー名とパスワードを確認できます。設定ファイルを作成する際に必要なので控えておきます。

![https://i.imgur.com/nc7yi1R.png](https://i.imgur.com/nc7yi1R.png)

### Docker で inadyn を起動する

まずは設定ファイルとなる `inadyn.conf` を作成します。

GitHub README に記載されている Google Domains の設定方法にしたがって作成します。

```conf:inadyn.conf
period     = 60 # 更新間隔
user-agent = Mozilla/5.0

provider domains.google.com {
    hostname = home.yumenomatayume.net # ホスト名を入れる
    username = fEawse3214DFAfda # ユーザー名を入れる
    password = fTfd123FDAasdf12 # パスワードを入れる
}
```

次に Docker で起動するための `docker-compose.yml` を作成します。

DockerHub に[公式イメージ](https://hub.docker.com/r/troglobit/inadyn)があるので、こちらを利用します。

```yaml:docker-compose.yml
---
version: "2.1"
services:
  inadyn:
    image: troglobit/inadyn:latest
    container_name: inadyn
    volumes:
      - type: bind
        source: "${PWD}/inadyn.conf"
        target: "/etc/inadyn.conf"
```

必要なファイルはこれだけです。それでは、`docker-compose up` を実行してみます。

```bash
❯ docker-compose up
[+] Running 1/0
 ⠿ Container inadyn  Created                                                                                                                                                                                                                                               0.0s
Attaching to inadyn
inadyn  | inadyn[1]: In-a-dyn version 2.9.1 -- Dynamic DNS update client.
inadyn  | inadyn[1]: Guessing DDNS plugin 'default@domains.google.com' from 'domains.google.com'
inadyn  | inadyn[1]: Update needed for alias home.yumenomatayume.net, new IP# 219.100.37.238
inadyn  | inadyn[1]: Updating cache for home.yumenomatayume.net
```

ドメインが登録されたようです。別ターミナルで名前解決できるか確認してみます。

```bash
❯ host home.yumenomatayume.net
home.yumenomatayume.net has address 219.100.37.238
```

名前解決できました。これを起動させておくだけで DNS が自動更新されるので、DDNS ってすごく便利ですね。

### kubernetes で inadyn を起動する

inadyn は自宅で常時稼働しているサーバーにデプロイしておきます。私の場合はおうち kubernetes クラスターにデプロイします。

まずはじめに、先ほどの `inadyn.conf` を `inadyn-secret` という名前で Secret リソースとして作成します。

```bash
kubectl create secret generic inadyn-secret --from-file=inadyn.conf=inadyn.conf
```

yaml にすると以下のようになります。

```yaml:inadyn-secret.yaml
apiVersion: v1
data:
  inadyn.conf: !secret # base64 でエンコードされた値が格納されます
kFqVTBrCn0K
kind: Secret
metadata:
  name: inadyn-secret
type: Opaque
```

次に Deployment リソースを作成します。以下のような yaml ファイルを作成します。

```yaml:inadyn.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inadyn
  labels:
    app: inadyn
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inadyn
  template:
    metadata:
      labels:
        app: inadyn
    spec:
      containers:
        - name: inadyn
          image: troglobit/inadyn:latest
          volumeMounts:
            - name: inadyn
              mountPath: /etc/inadyn.conf
              subPath: inadyn.conf
              readOnly: true
      volumes:
      - name: inadyn
        secret:
          secretName: inadyn-secret
          items:
            - key: inadyn.conf
              path: inadyn.conf
```

2 つのファイルを作成したら apply してみます。

```bash
❯ kubectl apply -f inadyn-secret.yaml -f inadyn.yaml
secret/inadyn-secret created
deployment.apps/inadyn created
```

ログを確認すると Docker Compose の時と同様に、DNS が更新されていることがわかります。

```bash
❯ kubectl logs -l app=inadyn
inadyn[1]: In-a-dyn version 2.9.1 -- Dynamic DNS update client.
inadyn[1]: Guessing DDNS plugin 'default@domains.google.com' from 'domains.google.com'
inadyn[1]: Update forced for alias home.yumenomatayume.net, new IP# 219.100.37.238
inadyn[1]: Updating cache for home.yumenomatayume.net
```

inadyn のデプロイは以上になります。

今回は自宅 DNS を自動更新してみましたが、オンプレミス環境などでグローバル IP を静的に管理している場合に活用できそうに感じました。

## その他

はじめは [ddclient](https://github.com/linuxserver/docker-ddclient) を検討していたのですが、公式コンテナイメージが軽量だった inadyn を選定しました。

```bash
# amd
❯ docker image ls
REPOSITORY                         TAG                    IMAGE ID       CREATED        SIZE
linuxserver/ddclient               latest                 14b44ffbe4af   3 weeks ago    78MB
troglobit/inadyn                   latest                 999faa5c6fb2   3 weeks ago    12.3MB

# arm
❯ docker image ls
REPOSITORY                         TAG                    IMAGE ID       CREATED         SIZE
lscr.io/linuxserver/ddclient       latest                 8a89c02a599b   3 weeks ago     83.5MB
troglobit/inadyn                   latest                 453a4cd7a3c5   3 weeks ago     12.1MB
```

[^1]: Internet Automated Dynamic DNS Client の略です。
[^2]: VPN Gate に接続して作業したため、実際の自宅 IP アドレスは記載のものと異なります。
