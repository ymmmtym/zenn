---
title: "ansible も asdf に対応していたのでインストールしてみた"
emoji: "⚙️"
type: "tech"
topics: [ansible, asdf]
published: true
---

ansible が adsf で管理されていたのでインストールしてみました。

https://github.com/amrox/asdf-pyapp

「asdf-pyapp」というリポジトリでからインストール可能で、他のアプリも管理されているようです。  
リポジトリの「Dependencies」にも記載の通り Python 3.6 以上を事前にインストールしておく必要があります。

## ローカル環境について

業務では Mac OS を利用しており、開発に利用するパッケージなどは（ほぼすべて）パッケージマネージャーで管理しています。  
パッケージマネージャーは 3 つ使っており、以下の優先度で管理しています。

- adsf
- aqua
- Homebrew

中には対応していないパッケージがあるので、asdf に対応していなければ aqua で管理する…みたいにしてます。

## ansible を asdf 経由でインストール

以前までは Homebrew で管理していた ansible ですが、asdf に対応していることを知ったのでそちらに乗り換えました。  
asdf では、`ansible-base` という名前で管理されています。

```bash
ASDF_PYAPP_INCLUDE_DEPS=1 asdf plugin-add ansible-base
asdf install ansible-base latest
asdf global ansible-base latest
```

最新バージョンをインストールしました。実行時のバージョンは `2.10.17` でした。

```bash
❯ ansible --version
ansible 2.10.17
  config file = None
  configured module search path = ['/Users/yumenomatayume/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /Users/yumenomatayume/.asdf/installs/ansible-base/2.10.17/venv/lib/python3.10/site-packages/ansible
  executable location = /Users/yumenomatayume/.asdf/installs/ansible-base/2.10.17/bin/ansible
  python version = 3.10.1 (main, Jan 12 2022, 21:01:44) [Clang 13.0.0 (clang-1300.0.29.30)]
```

## ansible コマンドの確認

利用可能なコマンドを調べたところ、頻繁に使いそうなものは網羅されているようです。  
（ansible-playbook、ansible-vault、ansible-doc など）

```bash
❯ ansible # tab 入力
ansible  (Define and run a single task 'playbook' against a set of hosts)  ansible-doc                                                       (Plugin documentation tool)  ansible-pull  (Pulls playbooks from a VCS repo and executes them for the local host)
ansible-config                               (View ansible configuration)  ansible-galaxy                       (Perform various Role and Collection related operations)  ansible-test                                                               (command)
ansible-connection                                              (command)  ansible-inventory                                                                      (None)  ansible-vault                 (Encryption/decryption utility for Ansible data files)
ansible-console                (REPL console for executing Ansible tasks)  ansible-playbook  (Runs Ansible playbooks, executing the defined tasks on the targeted hosts)
```

ちなみに、asdf では ansible-lint が管理されていないようです。  
（Homebrew では管理されていて、個人的にコミット前などによく使っていました。）

```bash
❯ brew search ansible
==> Formulae
ansible                                      ansible-cmdb                                 ansible-language-server                      ansible-lint                                 ansible@2.8                                  ansible@2.9

==> Casks
ansible-dk
```

ローカルでのテスト環境のツールは網羅されていないようですが、  
GitHub Actions などの CI を利用して、[ansible-lint](https://github.com/ansible/ansible-lint), [Molecule](https://ansible.readthedocs.io/projects/molecule/) などでテストする環境を整えると良さそうです。

ちなみに、[geerlingguy](https://github.com/geerlingguy) 氏のリポジトリには、Playbook や CI などのお手本のようなコード多く、とても参考になります。
