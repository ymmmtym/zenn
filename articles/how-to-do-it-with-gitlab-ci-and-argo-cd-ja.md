---
title: "GitOps in Kubernetes: How to do it with GitLab CI and Argo CD（和訳）"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "gitlab", "argocd", "gitops"]
published: false
---

:::message
この記事は、Andrzej Kaczynski 氏による [GitOps in Kubernetes: How to do it with GitLab CI and Argo CD](https://medium.com/@andrew.kaczynski/gitops-in-kubernetes-argo-cd-and-gitlab-ci-cd-5828c8eb34d6) の和訳です。
:::

最近の Cloud Native の世界では、GitOps の話題で持ちきりです。確かにこの継続的デリバリーのモデルは、現代の IT の世界では一種の革命です。GitOps が何なのかについては、インターネット上で多くのリソースを見つけることができますので、ここでは説明しません。最j初のグーグル検索結果にある記事の例では、それが何であるかが網羅的に説明されています。Git as a single source of truth for declarative infrastructure and application（宣言的なインフラとアプリケーションのための単一の真実の源としての Git）」という引用がもっとも分かりやすく私に訴えかけてきたので、これに従うこととします。

GitOps の定義を見つけるのは比較的簡単ですが、一方で、そのようなワークフローの例を見つけるのは非常に難しいことなのです。これが、私がこの物語を書いた最大の理由です。あなたの現在のインフラストラクチャのセットアップに GitOps を作成するのに役立つ多くの情報をここで見つけることができればと思います。

## TLDR

- Introduction
- Argo CD setup
- GitLab Project setup
- GitLab CI Pipeline
- How it works?
- Summary

## Introduction

下図は、今回紹介したGitOpsのワークフローを示したものです。

![Workflow](https://miro.medium.com/max/700/1*e4w0j0SUdsfx_U7hdHHpyw.png)

ここでは、GitLabとArgo CDが主役なので、この2つについて一言ずつ述べたいと思います。

ArgoCD は、Kubernetes のための宣言型 GitOps 継続的デリバリーツールです。Kubernetes の基本的な知識が必要ですが、比較的簡単に設定・使用できるので、私はこのツールを気に入っています。理解しやすい GUI が備わっています。私にとって重要だったのは、ArgoCD がいくつかの方法（kustomize、helm、ksonnet、あるいは jsonnet ファイル）でマニフェストをサポートしていることです。アプリケーションは、ArgoCD によって追加された専用のカスタムリソース定義の形で構成できます。これは、指定されたターゲット環境への希望するアプリケーション状態のデプロイメントを自動化するものです。アーキテクチャや利用可能な機能の詳細については、プロジェクトの公式ウェブサイトをご覧ください。

GitLabCI は、その名の通り GitLab が提供する継続的インテグレーションと継続的デリバリーのためのツールです。他の競合製品と比較すると、私にとっては CI/CD タスクのための非常に良い選択です。Jenkins のような他のツールと比較すると、パイプラインの作成が非常に簡単（.gitlab-ci.yaml ファイルがシンプル）なので、私はこのツールを気に入っています。このデモでは、必要な機能のほとんどをサポートしている無料版を使用するつもりです。では、さっそく始めてみましょう

## Argo CD Setup

これから、自分の PC でホストしている Minikube クラスターに GitOps をセットアップします。はじめに kubectl でクラスターにアクセスできることを確認します。私は通常、この目的のためにノードをリストアップします。

```bash
~> kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
m01    Ready    master   42s   v1.17.3
```

ArgoCD の GUI にアクセスするために ingress コントローラも必要です。Minikube で、追加のアドオンを有効にするだけです。

```bash
~> minikube addons enable ingress
```

次に、Helm（バージョン3が望ましい）を使って ArgoCD をインストールします。

```bash
~> kubectl create namespace argocd
~> helm repo add argo https://argoproj.github.io/argo-helm
~> helm install argocd -n argocd argo/argocd --values values.yaml
```

この例で私が使用したvalues.yamlファイルは以下のとおりです。

```yaml
appVersion: "1.4.2"
version: 1.8.7
server:
  ingress:
    enabled: true
    annotations: 
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
      nginx.ingress.kubernetes.io/ssl-passthrough: "true"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    hosts:
      - argocd.minikube.local
```

この後、私のArgo CDがインストールされます。アクセスするには、ingressのアドレスを取得し、ローカルの/etc/hostsに必要なエントリを追加します。

```bash
~> kubectl get ingress -n argocd
NAME            HOSTS                   ADDRESS          PORTS   AGE
argocd-server   argocd.minikube.local   192.168.99.105   80      15m
~> echo '192.168.99.105 argocd.minikube.local' | sudo tee -a /etc/hosts
```

これで、ブラウザからArgo CD GUIにアクセスできるようになりました。

![1*AbtlgpKVNcCgZXpkkSlpsg.png (700×425)](https://miro.medium.com/max/700/1*AbtlgpKVNcCgZXpkkSlpsg.png)

デフォルトのユーザー名はadmin、パスワードはポッドの名前です。例

```bash
~> kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
argocd-server-5cbcf6864-587hr
```

![1*bxyBj2No4sPBK8Ka_f-gMg.gif (934×711)](https://miro.medium.com/max/700/1*bxyBj2No4sPBK8Ka_f-gMg.gif)

とりあえずは以上です。次のステップでは、サンプルコードを含むGitLabプロジェクトを作成します。

## GitLab Project Setup

このデモのために、私はGoで書かれた小さなアプリケーションを持っています。これは、テキストメッセージとその横にポッド名を表示するウェブアプリです。全てはGitにあります。これがそのリポジトリです。

https://gitlab.com/andrew.kaczynski/gitops-webapp

GitLabはCI/CDだけでなく、プロジェクトごとにコンテナレジストリを提供しています。今回の例ではそれを利用したいと思います。
また、APIスコープを持つAccess Tokenを作成する必要があります。これは後のパイプラインでgitコミット時に使用されます。USER -> SETTINGS -> ACCESS TOKENSで作成することができます。
次に、プロジェクトに必要な環境変数を追加します（設定 -> CI/CD -> Variables）。

- CI_PUSH_TOKEN — the token
- CI_USERNAME — the username of the token owner

## ArgoCD Application Setup

いよいよGitOpsを使ってKubernetesにアプリケーションを設定する時が来ました。前にも述べたように、Argo CDには宣言的な設定を行うためのCRDが付属しています。もちろん、Infrastructure as a Codeを維持したいので、この方法が推奨されます。以下は、開発環境とプロード環境用のウェブアプリを構成するマニフェストです。

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-dev
  namespace: argocd
spec:
  project: default
  source: 
    repoURL: https://gitlab.com/andrew.kaczynski/gitops-webapp.git
    targetRevision: HEAD
    path: deployment/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-prod
  namespace: argocd
spec:
  project: default
  source: 
    repoURL: https://gitlab.com/andrew.kaczynski/gitops-webapp.git
    targetRevision: HEAD
    path: deployment/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
```

ここで最も重要なのは、その部分です。

- name — the name of our Argo CD application
- namespace — must be the same as Argo CD instance
- project — project name where the application will be configured (this is the way to organize your applications in Argo CD)
- repoURL— URL of our source code repository
- targetRevision — the git branch you want to use
- path — the path where Kubernetes manifests are stored inside repository
- destination — Kubernetes destination-related things (in this case the cluster is the same where Argo CD is hosted)

これらのマニフェストを他のマニフェストと同様にkubectlで適用します。クラスタに「application」タイプのオブジェクトが2つ作成されます。

```bash
~> kubectl get application -n argocd
NAME           AGE
web-app-dev    20m
web-app-prod   17m
```

Argo CD のウェブダッシュボードにアクセスすると、この2つのアプリケーションが表示されます。

![1*HrORJbzNdj03-J7NxjJ37A.gif (1306×711)](https://miro.medium.com/max/700/1*HrORJbzNdj03-J7NxjJ37A.gif)

そのうちの一つをクリックすると、アプリケーションとデプロイされたKubernetesオブジェクトの詳細が表示されます。gitops-webapp リポジトリの deployment/<env> ディレクトリ内にあるオブジェクトを確認できます。各フォルダ内に、kustomization.yamlを作成したことがわかります。Argo CD はこれを認識し、追加の設定なしに Kustomize を使って適用することができました!
このようなオプションもありますので、一度見てみることをお勧めします。GUIは非常に直感的なので、理解するのに問題はないでしょう。

## GitLab CI Pipeline

次のステップはパイプラインを作成することで、アプリケーションを自動的にビルドし、コンテナレジストリにプッシュし、必要な環境のKubernetesマニフェストを更新することができます。
以下の例は完璧とは言えませんが(テストが足りないなど)、GitOpsのワークフロー全体を示すものです。
基本的には、開発者は自分自身のブランチで作業しているということです。自分のブランチをコミットしてプッシュするたびに、ステージビルドが開始されます。開発者が自分の変更を master ブランチにマージすると、フルパイプラインが起動します。アプリケーションのビルド、コンテナ化、レジストリへのイメージのプッシュ、そしてステージに応じたKubernetesマニフェストの自動更新が行われます。また、Prodへのデプロイは、DevOpsの手動アクションが必要です。
GitLab CIにおけるパイプラインの定義は、プロジェクトのルートディレクトリにある.gitlab-ci.ymlファイルに格納される。
ファイルの先頭には、ステージの定義とデフォルトの環境変数、イメージやスクリプトステップの前があります。私のパイプラインからの抜粋です。

```yaml
stages:
- build
- publish
- deploy-dev
- deploy-prod
```

次にステージの定義と必要なタスクです。ここでのビルドプロセスは非常にシンプルです。dockerイメージのgolangの中で1つのコマンドを実行するだけです。そして、次のステージのためにアーティファクトを保存します。このステップは、どのブランチのどの新しいコミットに対しても機能します。

```yaml
build:
  stage: build
  image:
    name: golang:1.13.1
  script:
    - go build -o main main.go
  artifacts:
    paths:
      - main
  variables:
    CGO_ENABLED: 0
```

次の段階は、イメージをビルドしてGitLabレジストリにプッシュすることです。ここでは、Dockerエンジンにアクセスできないコンテナ内でイメージをビルドするため、Kanikoを使用しています。イメージは現在のコミットハッシュ (変数 $CI_COMMIT_HASH) を使ってビルドされ、GitLab のレジストリにプッシュされます。この段階は、master ブランチでの変更のみがトリガーされます。

```yaml
publish:
  stage: publish
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile ./Dockerfile --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  dependencies:
    - build  
  only:
    - master
```

次の段階は、アプリケーションを開発環境にデプロイすることです。GitOpsでは、Kubernetesのマニフェストを更新して、Argo CDが更新されたバージョンをプルして変更を適用できるようにすることを意味します。ここでは、masterブランチに変更をプッシュするために必要なユーザー名とトークンを持つプロジェクト用に定義された環境変数を使用しています。GitLab CI が [skip ci] というメッセージ付きのコミットを受け取ると、パイプラインはトリガーされません（Kustomize マニフェストに関する変更を公開するときに、再度パイプラインを実行したくありません）。

```yaml
deploy-dev:
  stage: deploy-dev
  image: alpine:3.8
  before_script:
    - apk add --no-cache git curl bash
    - curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
    - mv kustomize /usr/local/bin/
    - git remote set-url origin https://${CI_USERNAME}:${CI_PUSH_TOKEN}@gitlab.com/andrew.kaczynski/gitops-webapp.git
    - git config --global user.email "gitlab@gitlab.com"
    - git config --global user.name "GitLab CI/CD"
  script:
    - git checkout -B master
    - cd deployment/dev
    - kustomize edit set image $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - cat kustomization.yaml
    - git commit -am '[skip ci] DEV image update'
    - git push origin master
  only:
    - master
```

最後に、prod環境へのデプロイです。先ほどのものと非常に似ていますが、開始前に手動で操作する必要があります。

```yaml
deploy-prod:
  stage: deploy-prod
  image: alpine:3.8
  before_script:
    - apk add --no-cache git curl bash
    - curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
    - mv kustomize /usr/local/bin/
    - git remote set-url origin https://${CI_USERNAME}:${CI_PUSH_TOKEN}@gitlab.com/andrew.kaczynski/gitops-webapp.git
    - git config --global user.email "gitlab@gitlab.com"
    - git config --global user.name "GitLab CI/CD"
  script:
    - git checkout -B master
    - git pull origin master
    - cd deployment/prod
    - kustomize edit set image $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - cat kustomization.yaml
    - git commit -am '[skip ci] PROD image update'
    - git push origin master
  only:
    - master
  when: manual
```

これがパイプラインの完全な定義です。Jenkinsと比較して、GitLabでパイプラインを作成するのがいかに簡単かわかっていただけたでしょうか :)だから私はGitLabが大好きなんです

## How it works?

では、どのように連携していくかを見ていきましょう。まず、アプリケーションのイングレスの IP を取得し、ローカルの /etc/hosts ファイルに必要なエントリを追加します。

```bash
~> kubectl get ingress --all-namespaces |grep gitops
dev         gitops-webapp   webapp.dev.minikube.local    192.168.99.105   80      25m
prod        gitops-webapp   webapp.prod.minikube.local   192.168.99.105   80      25m
~> echo "192.168.99.105 webapp.dev.minikube.local" | sudo tee -a /etc/hosts
~> echo "192.168.99.105 webapp.prod.minikube.local" | sudo tee -a /etc/hosts
```

これで、ブラウザを使って、Webアプリケーションのメッセージを表示することができます。では、開発中のウェブアプリケーションを見てみましょう。

![1*7N-W_cwUR4hDfBkrDQRkYA.png (700×482)](https://miro.medium.com/max/700/1*7N-W_cwUR4hDfBkrDQRkYA.png)

そして、小さな変更を加えて、それをマスターにプッシュするときが来ました。main.go ファイルを編集して、変数 welcome の ANDREW という名前を GITOPS に変更するだけです。

```go
func main() {
   welcome := Welcome{"GITOPS", time.Now().Format(time.Stamp), os.Getenv("HOSTNAME")}
```

変更をコミットし、マスターにプッシュします。次に、GitLab project -> CI/CD -> Pipelinesに進みます。開始されたばかりの新しいパイプラインが表示されます。

![1*HKvr8FH804gh1QYON0oHSQ.png (700×96)](https://miro.medium.com/max/700/1*HKvr8FH804gh1QYON0oHSQ.png)

数分待つと、dev ステージへのデプロイが完了した時点で、ステータスがスキップされた新しいパイプラインが表示されます（パイプラインはすでに dev 用のマニフェストを更新しています）。自動同期モードのArgo CDは、1分以内にKubernetesの状態を更新します。Argo GUIで進捗を確認できます。更新されたメッセージを見るには、ウェブアプリを再度確認してください。

![1*WNd42lZYHvVSZxeQJX7nvQ.png (700×148)](https://miro.medium.com/max/700/1*WNd42lZYHvVSZxeQJX7nvQ.png)

![1*jzkDSpk4KmFM-arXxzJFIQ.png (700×459)](https://miro.medium.com/max/700/1*jzkDSpk4KmFM-arXxzJFIQ.png)

最後に、パイプラインステージを承認することで、手動でprodのビルドをトリガーします。その後、prodの画像も更新されます。

![1*MiJJMQylGhi6kMWLVir7DA.gif (1220×563)](https://miro.medium.com/max/700/1*MiJJMQylGhi6kMWLVir7DA.gif)

Syncの準備ができたときのArgo CDの更新ダッシュボードはこちらです。

![1*kfHC2zovxjI1QAgOwHvuhA.png (700×290)](https://miro.medium.com/max/700/1*kfHC2zovxjI1QAgOwHvuhA.png)

以上で、GitOps を使って開発環境とプロード環境へのデプロイに成功しました。

## Summary

私の投稿を楽しんで読んでいただけたなら幸いです。もしかしたら、いつかあなたのKubernetes環境がGitOpsに切り替わるかもしれませんね。もし何かコメントや質問があれば、いつでも喜んで相談に乗りますよ :-)
