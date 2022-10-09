---
title: "読書の秋には calibre で電子書籍を読もう"
emoji: "📚"
type: "idea"
topics: [calibre, calibreweb]
published: true
---

肌寒くなってきたので「今年は◯◯の秋を何にしようか」と考える季節になりました。

手始めに読書でもしようと思いましたが、電子書籍ファイルの管理が適当になっていて、購入したが読んでいない本があることに気づきました。（PDF や epub などの電子書籍は Google Drive にアップロードして、適当にフォルダー分けしています。）

そこで前々から気になっていた Calibre（カリバー）という電子書籍管理ソフトと、Calibre Web という Calibre の Web アプリケーションを使ってみました。

## Calibre をインストール

https://calibre-ebook.com/

Calibre は電子書籍管理ツールです。

Homebrew に対応していたので、以下のコマンドでインストールします。

```bash
brew install --casks calibre
```

インストールが完了したら、Calibre アプリを起動します。

初回起動時には、「ウェルカムウィザード」が開きます。

![calibre-welcome-wizard.png](/images/fall-of-calibre/calibre-welcome-wizard.png)
*calibre ウェルカムウィザード*

calibre ライブラリのフォルダーに、Calibre のデータベースや PDF などの電子書籍ファイルそのものが保存されます。

（上記の画像とは異なりますが）ライブラリの場所は、Mac にマウントさせている Google Drive のフォルダーを設定しました。

「ウェルカムウィザード」が完了すると、Calibre が起動します。

![calibre.png](/images/fall-of-calibre/calibre.png)
*calibre トップ画面*

あとは新しい電子書籍を買ったら「本を追加」から追加します。

PDF の場合は本の情報（タイトルや著者）が正確に読み込めない場合があるので、「書誌の編集」から適宜修正して管理します。タグなど設定して自分なりに管理しやすくしていきたいです。

:::message

「表示」をクリックするとビューアーが開くので、Calibre だけでも読書はできます。スマホなど**他の端末のブラウザ**から読みたい方は、calibre-web のインストールをオススメします。

:::

## Calibre Web をインストール

https://github.com/janeczku/calibre-web

calibre-web はブラウザ経由で calibre ライブラリにある電子書籍を読める Web アプリケーションです。

我が家には Synology 製の自宅 NAS があるので、Synology NAS の Docker 上で calibre-web を起動します。すでに Synology NAS と Google Drive は CloudSync で同期されています。
（直接 Google Drive にアクセスする機能もあるので、calibre-web のボリュームに calibre ライブラリがなくても閲覧できるようです。）

Docker アプリからぽちぽち起動させていきます。設定はデフォルトから以下の点を変更しました。

- ボリューム

| マウントするフォルダー | マウントパス | 説明 |
| --- | --- | --- |
| `docker/calibre-web/config` | `/config` | calibre-web の設定ファイルの永続化 |
| `share/cloud/google/Calibre` | `/books` | calibre ライブラリの保存先。共有フォルダーには guest ユーザー読み込み権限が必要です。（Docker コンテナー内で設定されるパーミッションのためです） |

- 環境変数

| 環境変数 | 値 |
| --- | --- |
| `LANGUAGE` | `ja_JP.UTF-8` |
| `LANG` | `ja_JP.UTF-8` |

![calibre-web-docker-on-synology.png](/images/fall-of-calibre/calibre-web-docker-on-synology.png)
*calibre-web Docker コンテナーの概要*

:::details Docker の設定ファイル

```json
{
    "CapAdd" : null,
    "CapDrop" : null,
    "cmd" : "",
    "cpu_priority" : 50,
    "enable_publish_all_ports" : false,
    "enable_restart_policy" : true,
    "enable_service_portal" : true,
    "enabled" : true,
    "env_variables" : [
        {
            "key" : "PATH",
            "value" : "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        },
        {
            "key" : "HOME",
            "value" : "/root"
        },
        {
            "key" : "LANGUAGE",
            "value" : "ja_JP.UTF-8"
        },
        {
            "key" : "LANG",
            "value" : "ja_JP.UTF-8"
        },
        {
            "key" : "TERM",
            "value" : "xterm"
        },
        {
            "key" : "S6_CMD_WAIT_FOR_SERVICES_MAXTIME",
            "value" : "0"
        }
    ],
    "exporting" : false,
    "id" : "7b81d0b71f09d165bc92e3c1db3193c14cf5ddd3a6ca3c065d9e37ad2b8e49ac",
    "image" : "linuxserver/calibre-web:latest",
    "is_ddsm" : false,
    "is_package" : false,
    "links" : [],
    "memory_limit" : 0,
    "name" : "calibre-web",
    "network" : [
        {
            "driver" : "bridge",
            "name" : "bridge"
        }
    ],
    "network_mode" : "bridge",
    "port_bindings" : [],
    "privileged" : false,
    "service_portals" : [
        {
            "port" : "8083",
            "protocol" : "http"
        }
    ],
    "services" : [
        {
            "display_name" : "Docker calibre-web 8083",
            "id" : "Docker-7b81d0b71f09d165bc92e3c1db3193c14cf5ddd3a6ca3c065d9e37ad2b8e49ac-8083",
            "proxy_target" : "http://172.17.0.2:8083",
            "service" : "Docker-7b81d0b71f09d165bc92e3c1db3193c14cf5ddd3a6ca3c065d9e37ad2b8e49ac-8083",
            "type" : "reverse_proxy"
        }
    ],
    "shortcut" : {
        "enable_shortcut" : false,
        "enable_status_page" : false,
        "enable_web_page" : false,
        "web_page_url" : ""
    },
    "use_host_network" : false,
    "volume_bindings" : [
        {
            "host_volume_file" : "/docker/calibre-web/config",
            "mount_point" : "/config",
            "type" : "rw"
        },
        {
            "host_volume_file" : "/share/cloud/google/Calibre",
            "mount_point" : "/books",
            "type" : "rw"
        }
    ]
}
:::

デフォルトではポート 8083 にバインドされるので、ブラウザで `<nas の IP アドレス>:8083` にアクセスするとログイン画面が表示されます。

![calibre-web-login.png](/images/fall-of-calibre/calibre-web-login.png)
*calibre-web ログイン画面*

初期ユーザ（`admin`）とパスワード（`admin123`）でログインします。

初回起動では Calibre ライブラリのパスを設定する必要があります。
「Location of Calibre Database」に Calibre ライブラリをマウントした `/books` を設定します。

![calibre-web-database-configuration.png](/images/fall-of-calibre/calibre-web-database-configuration.png)
*calibre-web calibre ライブラリ設定*

完了すると Calibre ライブラリにある電子書籍が表示されます。

![calibre-web-top.png](/images/fall-of-calibre/calibre-web-top.png)
*calibre-web トップページ*

適当な書籍を開いて、「Read in Browser」からブラウザ経由で閲覧できます。（初回は admin ユーザーに閲覧権限がないので、設定から適宜追加してください。）

![calibre-web-book-details.png](/images/fall-of-calibre/calibre-web-book-details.png)
*calibre-web 本の詳細*

![calibre-web-web-viewer.png](/images/fall-of-calibre/calibre-web-web-viewer.png)
*calibre-web Web ビューアー*

calibre-web にはユーザー管理やスケジュールタスクなどの機能があるようですが、使っていきながら設定していこうと思います。（詳細は割愛します。）

いいなと思ったのは本棚を作成できる機能があり、購読中の本などをまとめたり活用できそうです。

![calibre-web-shelf.png](/images/fall-of-calibre/calibre-web-shelf.png)
*calibre-web 本棚*

今年は calibre と calibre-web で読書します。calibre の秋！

## Reference（参考）

https://www.christian-knedel.de/ja/post/2020/february/20200213-synology-calibreweb/

直接 Google Drive にアクセスする方法は、GitHub の Wiki で紹介されています。

https://github.com/janeczku/calibre-web/wiki/G-Drive-Setup
