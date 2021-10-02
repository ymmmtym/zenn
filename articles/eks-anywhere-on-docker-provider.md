---
title: "EKS Anywhere（Docker プロバイダー）を試してみる"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eks", "kubernetes"]
published: false
---

https://anywhere.eks.amazonaws.com/

オンプレミス環境で EKS を試すことができる「EKS Anywhere」を使ってみます。

現在提供されているプロバイダーは、Docker（Kubernetes in Docker）と vSphere の 2 つのようです。

## 前提条件

クラスターを構築するマシンは以下の条件を満たす必要があります。

- Docker 20.x.x
- Mac OS (10.15) / Ubuntu (20.04.2 LTS)
- 4 CPU cores
- 16GB memory
- 30GB free disk space

今回はすでに Docker をインストールした **Ubuntu20.04.1 LTS** に、**Docker プロバイダー**を使って EKS Anywhere を作成します。[^1]

## eksctl のインストール

`eksctl` のバイナリをインストールします。

```bash
curl "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
    --silent --location \
    | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
```

`eksctl-anywhere` のバイナリをインストールします。

```bash
export EKSA_RELEASE="0.5.0" OS="$(uname -s | tr A-Z a-z)"
curl "https://anywhere-assets.eks.amazonaws.com/releases/eks-a/1/artifacts/eks-a/v${EKSA_RELEASE}/${OS}/eksctl-anywhere-v${EKSA_RELEASE}-${OS}-amd64.tar.gz" \
    --silent --location \
    | tar xz ./eksctl-anywhere
sudo mv ./eksctl-anywhere /usr/local/bin/
```

### インストールされたバージョンの確認

インストールされたバージョンを確認します。

```bash
$ eksctl anywhere version
v0.5.0
```

### kubectl のインストール

後ほど kubectl も使用するので、事前にインストールしておきます。

実施した環境は Ubuntu でしたので、[公式サイト](https://kubernetes.io/ja/docs/tasks/tools/install-kubectl/)に沿ってバイナリをインストールしました。

```bash
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### homebrew の場合

ちなみに、Mac に構築する場合は、homebrew よりインストールできます。

```bash
brew install aws/tap/eks-anywhere
```

## クラスターの構築

クラスターを定義する yaml ファイルを作成します。

```bash
CLUSTER_NAME=dev-cluster
eksctl anywhere generate clusterconfig $CLUSTER_NAME \
   --provider docker > $CLUSTER_NAME.yaml
```

クラスターを作成します。

```bash
eksctl anywhere create cluster -f $CLUSTER_NAME.yaml
```

少し待って作成が完了すると、以下のファイル構成になっています。

```bash
kube@eks-ubuntu:~$ tree
.
├── dev-cluster
│   ├── dev-cluster-eks-a-cluster.kubeconfig
│   └── dev-cluster-eks-a-cluster.yaml
└── dev-cluster.yaml

1 directory, 3 files
```

`KUBECONFIG` を読み込んで、作成したクラスターに対して `kubectl` コマンドを実行できるようにします。

```bash
export KUBECONFIG=${PWD}/${CLUSTER_NAME}/${CLUSTER_NAME}-eks-a-cluster.kubeconfig
```

作成されたリソース（namespace, nodes, pods）は以下のようになっていました。

```bash
kube@eks-ubuntu:~$ kubectl get ns,nodes,pods -A
NAME                                          STATUS   AGE
namespace/capd-system                         Active   5m12s
namespace/capi-kubeadm-bootstrap-system       Active   5m18s
namespace/capi-kubeadm-control-plane-system   Active   5m14s
namespace/capi-system                         Active   5m19s
namespace/capi-webhook-system                 Active   5m21s
namespace/cert-manager                        Active   6m7s
namespace/default                             Active   6m59s
namespace/eksa-system                         Active   4m19s
namespace/etcdadm-bootstrap-provider-system   Active   5m17s
namespace/etcdadm-controller-system           Active   5m17s
namespace/kube-node-lease                     Active   7m
namespace/kube-public                         Active   7m
namespace/kube-system                         Active   7m

NAME                                     STATUS   ROLES                  AGE   VERSION
node/dev-cluster-md-0-6c47765b4d-dvt7k   Ready    <none>                 4m   v1.21.2-eks-1-21-4
node/dev-cluster-mmhgm                   Ready    control-plane,master   4m   v1.21.2-eks-1-21-4

NAMESPACE                           NAME                                                                 READY   STATUS    RESTARTS   AGE
capd-system                         pod/capd-controller-manager-659dd5f8bc-g62j7                         2/2     Running   0          5m12s
capi-kubeadm-bootstrap-system       pod/capi-kubeadm-bootstrap-controller-manager-69889cb844-h7z7v       2/2     Running   0          5m18s
capi-kubeadm-control-plane-system   pod/capi-kubeadm-control-plane-controller-manager-6ddc66fb75-7nfzw   2/2     Running   0          5m14s
capi-system                         pod/capi-controller-manager-db59f5789-jrfj7                          2/2     Running   0          5m19s
capi-webhook-system                 pod/capi-controller-manager-64b8c548db-jgz8b                         2/2     Running   0          5m19s
capi-webhook-system                 pod/capi-kubeadm-bootstrap-controller-manager-68b8cc9759-t2s5d       2/2     Running   0          5m18s
capi-webhook-system                 pod/capi-kubeadm-control-plane-controller-manager-7dc88f767d-krbll   2/2     Running   0          5m15s
cert-manager                        pod/cert-manager-5f6b885b4-lvwtr                                     1/1     Running   0          6m6s
cert-manager                        pod/cert-manager-cainjector-bb6d9bcb5-tssvc                          1/1     Running   0          6m6s
cert-manager                        pod/cert-manager-webhook-56cbc8f5b8-xb76s                            1/1     Running   0          6m6s
eksa-system                         pod/eksa-controller-manager-6769764b45-mrshh                         2/2     Running   0          4m15s
etcdadm-bootstrap-provider-system   pod/etcdadm-bootstrap-provider-controller-manager-54476b7bf9-drdh8   2/2     Running   0          5m17s
etcdadm-controller-system           pod/etcdadm-controller-controller-manager-d5795556-49hw4             2/2     Running   0          5m16s
kube-system                         pod/cilium-csvk8                                                     1/1     Running   0          6m8s
kube-system                         pod/cilium-jqz98                                                     1/1     Running   0          6m8s
kube-system                         pod/cilium-operator-6bf46cc6c6-gtxjx                                 1/1     Running   0          6m8s
kube-system                         pod/cilium-operator-6bf46cc6c6-sjg9c                                 1/1     Running   0          6m8s
kube-system                         pod/coredns-7c68f85774-7j9rv                                         1/1     Running   0          6m39s
kube-system                         pod/coredns-7c68f85774-sflhf                                         1/1     Running   0          6m39s
kube-system                         pod/kube-apiserver-dev-cluster-mmhgm                                 1/1     Running   0          6m47s
kube-system                         pod/kube-controller-manager-dev-cluster-mmhgm                        1/1     Running   0          6m28s
kube-system                         pod/kube-proxy-5td77                                                 1/1     Running   0          6m11s
kube-system                         pod/kube-proxy-xv6s8                                                 1/1     Running   0          6m39s
kube-system                         pod/kube-scheduler-dev-cluster-mmhgm                                 1/1     Running   0          6m28s
```

デフォルトでは、2 つの node が作成されました。

実際に作成されたコンテナーは以下のようになっていました。

```bash
kube@eks-ubuntu:~$ docker container ls
CONTAINER ID   IMAGE                                                                                COMMAND                  CREATED          STATUS          PORTS                                  NAMES
559fe86b58f4   public.ecr.aws/eks-anywhere/kubernetes-sigs/kind/node:v1.21.2-eks-d-1-21-4-eks-a-1   "/usr/local/bin/entr…"   51 minutes ago   Up 51 minutes                                          dev-cluster-md-0-6c47765b4d-dvt7k
c127e82afea4   public.ecr.aws/eks-anywhere/kubernetes-sigs/kind/node:v1.21.2-eks-d-1-21-4-eks-a-1   "/usr/local/bin/entr…"   52 minutes ago   Up 52 minutes   38859/tcp, 127.0.0.1:38859->6443/tcp   dev-cluster-mmhgm
15b9d39e5a7a   public.ecr.aws/eks-anywhere/kubernetes-sigs/kind/node:v1.21.2-eks-d-1-21-4-eks-a-1   "/usr/local/bin/entr…"   52 minutes ago   Up 52 minutes                                          dev-cluster-etcd-sn9hc
d2baa02274d9   kindest/haproxy:v20210715-a6da3463                                                   "haproxy -sf 7 -W -d…"   52 minutes ago   Up 52 minutes   34661/tcp, 0.0.0.0:34661->6443/tcp     dev-cluster-lb

```

## テストアプリのデプロイ

公式ドキュメントに用意されたアプリをデプロイします。

``` bash
kube@eks-ubuntu:~$ kubectl apply -f "https://anywhere.eks.amazonaws.com/manifests/hello-eks-a.yaml"
deployment.apps/hello-eks-a created
service/hello-eks-a created
```

アプリケーションの状態を確認します。

```bash
kube@eks-ubuntu:~$ kubectl get -f "https://anywhere.eks.amazonaws.com/manifests/hello-eks-a.yaml"
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-eks-a   1/1     1            1           2m45s

NAME                  TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/hello-eks-a   NodePort   10.135.128.114   <none>        80:30567/TCP   2m45s
```

NodePort で公開されているので、クラスター（Docker）の IP アドレスを確認します。

```bash
kube@eks-ubuntu:~$ kubectl get nodes -o wide
NAME                                STATUS   ROLES                  AGE   VERSION              INTERNAL-IP   EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION     CONTAINER-RUNTIME
dev-cluster-md-0-6c47765b4d-dvt7k   Ready    <none>                 23m   v1.21.2-eks-1-21-4   172.18.0.6    <none>        Amazon Linux 2   5.4.0-42-generic   containerd://1.4.6
dev-cluster-mmhgm                   Ready    control-plane,master   24m   v1.21.2-eks-1-21-4   172.18.0.5    <none>        Amazon Linux 2   5.4.0-42-generic   containerd://1.4.6
```

`172.18.0.{5,6}` の 2 つでクラスターが組まれていました。どちらかの IP アドレスの公開された NodePort に対して curl を実行するとアプリがデプロイされていることを確認できます。

```bash
kube@eks-ubuntu:~$ curl 172.18.0.5:30567
⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢

Thank you for using

███████╗██╗  ██╗███████╗                                             
██╔════╝██║ ██╔╝██╔════╝                                             
█████╗  █████╔╝ ███████╗                                             
██╔══╝  ██╔═██╗ ╚════██║                                             
███████╗██║  ██╗███████║                                             
╚══════╝╚═╝  ╚═╝╚══════╝                                             
                                                                     
 █████╗ ███╗   ██╗██╗   ██╗██╗    ██╗██╗  ██╗███████╗██████╗ ███████╗
██╔══██╗████╗  ██║╚██╗ ██╔╝██║    ██║██║  ██║██╔════╝██╔══██╗██╔════╝
███████║██╔██╗ ██║ ╚████╔╝ ██║ █╗ ██║███████║█████╗  ██████╔╝█████╗  
██╔══██║██║╚██╗██║  ╚██╔╝  ██║███╗██║██╔══██║██╔══╝  ██╔══██╗██╔══╝  
██║  █║██║ ╚████║   ██║   ╚███╔███╔╝██║  ██║███████╗██║  ██║███████╗
╚═╝  ╚═╝╚═╝  ╚═══╝   ╚═╝    ╚══╝╚══╝ ╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝╚══════╝
                                                                     
You have successfully deployed the hello-eks-a pod hello-eks-a-9644dd8dc-kgkpt

For more information check out
https://anywhere.eks.amazonaws.com

⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢⬡⬢
```

[^1]: 公式ドキュメントでは Ubuntu20.04.2LTS が必要と記載されてましたが、1LTS でも動作しました。
