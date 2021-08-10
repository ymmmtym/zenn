---
title: "RKE入門 ~CLIからterraform管理まで~"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes", "rke", "rancher", "terraform"]
published: true
---

https://rancher.com/docs/rke/latest/en/

今更ながら、RKE(Rancher Kubernetes Engine)という Kubernetes クラスタのデプロイツールを知りました。

# 前置き

これまで ansible の使用経験があったので、Kubespray,kubeadm を使ってクラスタを構築していました。

- kubespray

https://github.com/kubernetes-sigs/kubespray

- kubeadm
  - 以下は kubeadm で kubernetes クラスタを構築する Ansible Role です

https://github.com/geerlingguy/ansible-role-kubernetes

これらのツールと比べて、RKE を使用すると簡単にクラスタ構築が可能で学習コストも低いです。
さらに Terraform Provider も提供されており、IaC しやすいといったメリットもあります。

以下の公式サイトには、Kops,Kubespray と比べて Rancher が優れている点がまとまっています。

https://www.rancher.co.jp/what-is-rancher/how-is-rancher-different/

# k8s クラスタ構築

まずは最も一般的な、k8s クラスタ用の yml ファイルを作成して、CLI でクラスタ構築してみます。

以下の node を用意して構築します。(こちらは全て ESXi 上にデプロイされています)

| ホスト名      | role   | 外部IPアドレス | 内部IPアドレス | SSHログイン名 | SSH Private Key    |
| ------------- | ------ | -------------- | -------------- | ------------- | ------------------ |
| k8s-mast er01 | master | 192.168.100.4  | 192.168.101.4  | kube          | ~/.ssh/id_rsa.kube |
| k8s-worker01  | worker | 192.168.100.5  | 192.168.101.5  | kube          | ~/.ssh/id_rsa.kube |
| k8s-worker02  | worker | 192.168.100.6  | 192.168.101.6  | kube          | ~/.ssh/id_rsa.kube |

## 前提条件

- rke 実行端末は MAC
- node は全て Ubuntu 20.04
- network は flannel を使用
- 外部 IP アドレスと MAC から node に SSH 鍵認証でログイン可能
- 内部 IP アドレスでクラスタ内が通信する(外部 IP と同一でも問題ありません)

その他の細かい条件は RKE のデフォルトを使用します。

## MacにRKEのインストール

Mac に RKE をインストールします。brew からインストールできます。

```bash
brew install rke
```

## 各nodeの設定

各 node が k8s クラスタとして動作するための必要な要件が、以下の前提条件に記載されています。

https://rancher.com/docs/rke/latest/en/os/

基本的には kubeadm と同様の要件が必要になります。私の環境では、以下を各 node で実施する必要がありました。

- swap の無効化
- Docker インストール & 実行ユーザ設定
- iptables の設定

### swap の無効化

まずは swap を無効化するために以下のコマンドを実行します。

```bash
sudo swapoff -a
```

### Docker インストール & 実行ユーザ設定

次に、各クラスタには Docker をインストールしておき、実行ユーザが sudo なしで docker コマンドを実行できるようにしておきます。

Rancher には Docker をインストールできるシェルスクリプトが提供されていて、簡単にインストールできます。

```bash
curl https://releases.rancher.com/install-docker/20.10.sh | sh
sudo usermod -a -G docker kube
```

### iptables の設定

最後に iptables(firewall) の設定です。今回は検証環境なので全ての通信を許可しています。

```bash
sudo iptables -F
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo netfilter-persistent save
```

:::message
先ほどの前提条件に k8s クラスタが使用するポートについて細かく記載されているので、設定したい方は参照してみてください。
:::

## k8s クラスタ構築

https://rancher.com/docs/rke/latest/en/installation/#prepare-the-nodes-for-the-kubernetes-cluster

まずクラスタの設定ファイルである `cluster.yml` の作成を作成します。

```bash
rke config --name cluster.yml
```

クラスタの IP アドレスや秘密鍵、コンテナランタイムなどの設定をします。

以下のように対話形式で入力していくと、`cluster.yml` を作成できます。(クラスタネットワークの CIDR など、デフォルトで問題ないものはそのまま <Enter> を押しています。)

:::details 実行ログはこちらです。

```bash
$ rke config --name cluster.yml
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]: ~/.ssh/id_rsa.kube
[+] Number of Hosts [1]: 3
[+] SSH Address of host (1) [none]: 192.168.100.4
[+] SSH Port of host (1) [22]: 22
[+] SSH Private Key Path of host (192.168.100.4) [none]: ~/.ssh/id_rsa.kube
[+] SSH User of host (192.168.100.4) [ubuntu]: kube
[+] Is host (192.168.100.4) a Control Plane host (y/n)? [y]: y
[+] Is host (192.168.100.4) a Worker host (y/n)? [n]: y
[+] Is host (192.168.100.4) an etcd host (y/n)? [n]: y
[+] Override Hostname of host (192.168.100.4) [none]: k8s-master01
[+] Internal IP of host (192.168.100.4) [none]: 192.168.101.4
[+] Docker socket path on host (192.168.100.4) [/var/run/docker.sock]: 
[+] SSH Address of host (2) [none]: 192.168.100.5
[+] SSH Port of host (2) [22]: 
[+] SSH Private Key Path of host (192.168.100.5) [none]: ~/.ssh/id_rsa.kube
[+] SSH User of host (192.168.100.5) [ubuntu]: kube
[+] Is host (192.168.100.5) a Control Plane host (y/n)? [y]: n
[+] Is host (192.168.100.5) a Worker host (y/n)? [n]: y
[+] Is host (192.168.100.5) an etcd host (y/n)? [n]: n
[+] Override Hostname of host (192.168.100.5) [none]: k8s-worker01
[+] Internal IP of host (192.168.100.5) [none]: 192.168.101.5
[+] Docker socket path on host (192.168.100.5) [/var/run/docker.sock]: 
[+] SSH Address of host (3) [none]: 192.168.100.6
[+] SSH Port of host (3) [22]: 
[+] SSH Private Key Path of host (192.168.100.6) [none]: ~/.ssh/id_rsa.kube
[+] SSH User of host (192.168.100.6) [ubuntu]: kube
[+] Is host (192.168.100.6) a Control Plane host (y/n)? [y]: n
[+] Is host (192.168.100.6) a Worker host (y/n)? [n]: y
[+] Is host (192.168.100.6) an etcd host (y/n)? [n]: n
[+] Override Hostname of host (192.168.100.6) [none]: k8s-worker02
[+] Internal IP of host (192.168.100.6) [none]: 192.168.101.6
[+] Docker socket path on host (192.168.100.6) [/var/run/docker.sock]: 
[+] Network Plugin Type (flannel, calico, weave, canal, aci) [canal]: flannel
[+] Authentication Strategy [x509]: 
[+] Authorization Mode (rbac, none) [rbac]: 
[+] Kubernetes Docker image [rancher/hyperkube:v1.20.8-rancher1]: 
[+] Cluster domain [cluster.local]: 
[+] Service Cluster IP Range [10.43.0.0/16]: 
[+] Enable PodSecurityPolicy [n]: 
[+] Cluster Network CIDR [10.42.0.0/16]: 
[+] Cluster DNS Service IP [10.43.0.10]: 
[+] Add addon manifest URLs or YAML files [no]: 

$ ls
cluster.yml
```

:::

この `rke config` は、`cluster.yml` を作成するコマンドなので、手作業で作っても問題ありません。

以下に yml ファイルのオプションが記載されているので、ベースとなる `cluster.yml` を作成したら、要件をもとに修正していくのが良さそうです。

https://rancher.com/docs/rke/latest/en/config-options

準備ができたので k8s クラスタを作成します。以下のコマンドを実行するだけです。

```bash
rke up
```

:::details 実行ログはこちらです。

```bash
$ rke up
INFO[0000] Running RKE version: v1.2.9                  
INFO[0000] Initiating Kubernetes cluster                
INFO[0000] [certificates] GenerateServingCertificate is disabled, checking if there are unused kubelet certificates 
INFO[0000] [certificates] Generating admin certificates and kubeconfig 
INFO[0000] Successfully Deployed state file at [./cluster.rkestate] 
INFO[0000] Building Kubernetes cluster                  
INFO[0000] [dialer] Setup tunnel for host [192.168.100.6] 
INFO[0000] [dialer] Setup tunnel for host [192.168.100.4] 
INFO[0000] [dialer] Setup tunnel for host [192.168.100.5] 
INFO[0000] [network] Deploying port listener containers 
INFO[0000] Pulling image [rancher/rke-tools:v0.1.75] on host [192.168.100.4], try #1 
INFO[0016] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0017] Starting container [rke-etcd-port-listener] on host [192.168.100.4], try #1 
INFO[0017] [network] Successfully started [rke-etcd-port-listener] container on host [192.168.100.4] 
INFO[0017] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0017] Starting container [rke-cp-port-listener] on host [192.168.100.4], try #1 
INFO[0017] [network] Successfully started [rke-cp-port-listener] container on host [192.168.100.4] 
INFO[0017] Pulling image [rancher/rke-tools:v0.1.75] on host [192.168.100.6], try #1 
INFO[0017] Pulling image [rancher/rke-tools:v0.1.75] on host [192.168.100.5], try #1 
INFO[0017] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0017] Starting container [rke-worker-port-listener] on host [192.168.100.4], try #1 
INFO[0017] [network] Successfully started [rke-worker-port-listener] container on host [192.168.100.4] 
INFO[0030] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0030] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0030] Starting container [rke-worker-port-listener] on host [192.168.100.6], try #1 
INFO[0030] Starting container [rke-worker-port-listener] on host [192.168.100.5], try #1 
INFO[0031] [network] Successfully started [rke-worker-port-listener] container on host [192.168.100.6] 
INFO[0031] [network] Successfully started [rke-worker-port-listener] container on host [192.168.100.5] 
INFO[0031] [network] Port listener containers deployed successfully 
INFO[0031] [network] Running control plane -> etcd port checks 
INFO[0031] [network] Checking if host [192.168.100.4] can connect to host(s) [192.168.101.4] on port(s) [2379], try #1 
INFO[0031] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0031] Starting container [rke-port-checker] on host [192.168.100.4], try #1 
INFO[0031] [network] Successfully started [rke-port-checker] container on host [192.168.100.4] 
INFO[0031] Removing container [rke-port-checker] on host [192.168.100.4], try #1 
INFO[0031] [network] Running control plane -> worker port checks 
INFO[0031] [network] Checking if host [192.168.100.4] can connect to host(s) [192.168.101.4 192.168.101.5 192.168.101.6] on port(s) [10250], try #1 
INFO[0031] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0031] Starting container [rke-port-checker] on host [192.168.100.4], try #1 
INFO[0031] [network] Successfully started [rke-port-checker] container on host [192.168.100.4] 
INFO[0031] Removing container [rke-port-checker] on host [192.168.100.4], try #1 
INFO[0031] [network] Running workers -> control plane port checks 
INFO[0031] [network] Checking if host [192.168.100.5] can connect to host(s) [192.168.101.4] on port(s) [6443], try #1 
INFO[0031] [network] Checking if host [192.168.100.4] can connect to host(s) [192.168.101.4] on port(s) [6443], try #1 
INFO[0031] [network] Checking if host [192.168.100.6] can connect to host(s) [192.168.101.4] on port(s) [6443], try #1 
INFO[0031] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0031] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0031] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0031] Starting container [rke-port-checker] on host [192.168.100.4], try #1 
INFO[0031] Starting container [rke-port-checker] on host [192.168.100.6], try #1 
INFO[0031] Starting container [rke-port-checker] on host [192.168.100.5], try #1 
INFO[0031] [network] Successfully started [rke-port-checker] container on host [192.168.100.4] 
INFO[0031] Removing container [rke-port-checker] on host [192.168.100.4], try #1 
INFO[0031] [network] Successfully started [rke-port-checker] container on host [192.168.100.5] 
INFO[0031] [network] Successfully started [rke-port-checker] container on host [192.168.100.6] 
INFO[0031] Removing container [rke-port-checker] on host [192.168.100.5], try #1 
INFO[0031] Removing container [rke-port-checker] on host [192.168.100.6], try #1 
INFO[0031] [network] Checking KubeAPI port Control Plane hosts 
INFO[0031] [network] Removing port listener containers  
INFO[0031] Removing container [rke-etcd-port-listener] on host [192.168.100.4], try #1 
INFO[0031] [remove/rke-etcd-port-listener] Successfully removed container on host [192.168.100.4] 
INFO[0031] Removing container [rke-cp-port-listener] on host [192.168.100.4], try #1 
INFO[0032] [remove/rke-cp-port-listener] Successfully removed container on host [192.168.100.4] 
INFO[0032] Removing container [rke-worker-port-listener] on host [192.168.100.4], try #1 
INFO[0032] Removing container [rke-worker-port-listener] on host [192.168.100.5], try #1 
INFO[0032] Removing container [rke-worker-port-listener] on host [192.168.100.6], try #1 
INFO[0032] [remove/rke-worker-port-listener] Successfully removed container on host [192.168.100.6] 
INFO[0032] [remove/rke-worker-port-listener] Successfully removed container on host [192.168.100.5] 
INFO[0032] [remove/rke-worker-port-listener] Successfully removed container on host [192.168.100.4] 
INFO[0032] [network] Port listener containers removed successfully 
INFO[0032] [certificates] Deploying kubernetes certificates to Cluster nodes 
INFO[0032] Checking if container [cert-deployer] is running on host [192.168.100.5], try #1 
INFO[0032] Checking if container [cert-deployer] is running on host [192.168.100.6], try #1 
INFO[0032] Checking if container [cert-deployer] is running on host [192.168.100.4], try #1 
INFO[0032] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0032] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0032] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0032] Starting container [cert-deployer] on host [192.168.100.6], try #1 
INFO[0032] Starting container [cert-deployer] on host [192.168.100.4], try #1 
INFO[0032] Starting container [cert-deployer] on host [192.168.100.5], try #1 
INFO[0032] Checking if container [cert-deployer] is running on host [192.168.100.6], try #1 
INFO[0032] Checking if container [cert-deployer] is running on host [192.168.100.4], try #1 
INFO[0032] Checking if container [cert-deployer] is running on host [192.168.100.5], try #1 
INFO[0037] Checking if container [cert-deployer] is running on host [192.168.100.6], try #1 
INFO[0037] Removing container [cert-deployer] on host [192.168.100.6], try #1 
INFO[0037] Checking if container [cert-deployer] is running on host [192.168.100.4], try #1 
INFO[0037] Removing container [cert-deployer] on host [192.168.100.4], try #1 
INFO[0037] Checking if container [cert-deployer] is running on host [192.168.100.5], try #1 
INFO[0037] Removing container [cert-deployer] on host [192.168.100.5], try #1 
INFO[0037] [reconcile] Rebuilding and updating local kube config 
INFO[0037] Successfully Deployed local admin kubeconfig at [./kube_config_cluster.yml] 
WARN[0037] [reconcile] host [192.168.100.4] is a control plane node without reachable Kubernetes API endpoint in the cluster 
WARN[0037] [reconcile] no control plane node with reachable Kubernetes API endpoint in the cluster found 
INFO[0037] [certificates] Successfully deployed kubernetes certificates to Cluster nodes 
INFO[0037] [file-deploy] Deploying file [/etc/kubernetes/audit-policy.yaml] to node [192.168.100.4] 
INFO[0037] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0037] Starting container [file-deployer] on host [192.168.100.4], try #1 
INFO[0037] Successfully started [file-deployer] container on host [192.168.100.4] 
INFO[0037] Waiting for [file-deployer] container to exit on host [192.168.100.4] 
INFO[0037] Waiting for [file-deployer] container to exit on host [192.168.100.4] 
INFO[0037] Container [file-deployer] is still running on host [192.168.100.4]: stderr: [], stdout: [] 
INFO[0038] Waiting for [file-deployer] container to exit on host [192.168.100.4] 
INFO[0038] Removing container [file-deployer] on host [192.168.100.4], try #1 
INFO[0038] [remove/file-deployer] Successfully removed container on host [192.168.100.4] 
INFO[0038] [/etc/kubernetes/audit-policy.yaml] Successfully deployed audit policy file to Cluster control nodes 
INFO[0038] [reconcile] Reconciling cluster state        
INFO[0038] [reconcile] This is newly generated cluster  
INFO[0038] Pre-pulling kubernetes images                
INFO[0038] Pulling image [rancher/hyperkube:v1.20.8-rancher1] on host [192.168.100.4], try #1 
INFO[0038] Pulling image [rancher/hyperkube:v1.20.8-rancher1] on host [192.168.100.5], try #1 
INFO[0038] Pulling image [rancher/hyperkube:v1.20.8-rancher1] on host [192.168.100.6], try #1 
INFO[0121] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.6] 
INFO[0126] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.4] 
INFO[0128] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.5] 
INFO[0128] Kubernetes images pulled successfully        
INFO[0128] [etcd] Building up etcd plane..              
INFO[0128] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0128] Starting container [etcd-fix-perm] on host [192.168.100.4], try #1 
INFO[0129] Successfully started [etcd-fix-perm] container on host [192.168.100.4] 
INFO[0129] Waiting for [etcd-fix-perm] container to exit on host [192.168.100.4] 
INFO[0129] Waiting for [etcd-fix-perm] container to exit on host [192.168.100.4] 
INFO[0129] Container [etcd-fix-perm] is still running on host [192.168.100.4]: stderr: [], stdout: [] 
INFO[0130] Waiting for [etcd-fix-perm] container to exit on host [192.168.100.4] 
INFO[0130] Removing container [etcd-fix-perm] on host [192.168.100.4], try #1 
INFO[0130] [remove/etcd-fix-perm] Successfully removed container on host [192.168.100.4] 
INFO[0130] Pulling image [rancher/mirrored-coreos-etcd:v3.4.15-rancher1] on host [192.168.100.4], try #1 
INFO[0137] Image [rancher/mirrored-coreos-etcd:v3.4.15-rancher1] exists on host [192.168.100.4] 
INFO[0137] Starting container [etcd] on host [192.168.100.4], try #1 
INFO[0137] [etcd] Successfully started [etcd] container on host [192.168.100.4] 
INFO[0137] [etcd] Running rolling snapshot container [etcd-snapshot-once] on host [192.168.100.4] 
INFO[0137] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0137] Starting container [etcd-rolling-snapshots] on host [192.168.100.4], try #1 
INFO[0137] [etcd] Successfully started [etcd-rolling-snapshots] container on host [192.168.100.4] 
INFO[0142] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0142] Starting container [rke-bundle-cert] on host [192.168.100.4], try #1 
INFO[0142] [certificates] Successfully started [rke-bundle-cert] container on host [192.168.100.4] 
INFO[0142] Waiting for [rke-bundle-cert] container to exit on host [192.168.100.4] 
INFO[0142] Container [rke-bundle-cert] is still running on host [192.168.100.4]: stderr: [], stdout: [] 
INFO[0143] Waiting for [rke-bundle-cert] container to exit on host [192.168.100.4] 
INFO[0143] [certificates] successfully saved certificate bundle [/opt/rke/etcd-snapshots//pki.bundle.tar.gz] on host [192.168.100.4] 
INFO[0143] Removing container [rke-bundle-cert] on host [192.168.100.4], try #1 
INFO[0143] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0143] Starting container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0144] [etcd] Successfully started [rke-log-linker] container on host [192.168.100.4] 
INFO[0144] Removing container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0144] [remove/rke-log-linker] Successfully removed container on host [192.168.100.4] 
INFO[0144] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0144] Starting container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0144] [etcd] Successfully started [rke-log-linker] container on host [192.168.100.4] 
INFO[0144] Removing container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0144] [remove/rke-log-linker] Successfully removed container on host [192.168.100.4] 
INFO[0144] [etcd] Successfully started etcd plane.. Checking etcd cluster health 
INFO[0144] [etcd] etcd host [192.168.100.4] reported healthy=true 
INFO[0144] [controlplane] Building up Controller Plane.. 
INFO[0144] Checking if container [service-sidekick] is running on host [192.168.100.4], try #1 
INFO[0144] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0144] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.4] 
INFO[0144] Starting container [kube-apiserver] on host [192.168.100.4], try #1 
INFO[0144] [controlplane] Successfully started [kube-apiserver] container on host [192.168.100.4] 
INFO[0144] [healthcheck] Start Healthcheck on service [kube-apiserver] on host [192.168.100.4] 
INFO[0152] [healthcheck] service [kube-apiserver] on host [192.168.100.4] is healthy 
INFO[0152] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0152] Starting container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0152] [controlplane] Successfully started [rke-log-linker] container on host [192.168.100.4] 
INFO[0152] Removing container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0153] [remove/rke-log-linker] Successfully removed container on host [192.168.100.4] 
INFO[0153] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.4] 
INFO[0153] Starting container [kube-controller-manager] on host [192.168.100.4], try #1 
INFO[0153] [controlplane] Successfully started [kube-controller-manager] container on host [192.168.100.4] 
INFO[0153] [healthcheck] Start Healthcheck on service [kube-controller-manager] on host [192.168.100.4] 
INFO[0158] [healthcheck] service [kube-controller-manager] on host [192.168.100.4] is healthy 
INFO[0158] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0158] Starting container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0158] [controlplane] Successfully started [rke-log-linker] container on host [192.168.100.4] 
INFO[0158] Removing container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0158] [remove/rke-log-linker] Successfully removed container on host [192.168.100.4] 
INFO[0158] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.4] 
INFO[0158] Starting container [kube-scheduler] on host [192.168.100.4], try #1 
INFO[0159] [controlplane] Successfully started [kube-scheduler] container on host [192.168.100.4] 
INFO[0159] [healthcheck] Start Healthcheck on service [kube-scheduler] on host [192.168.100.4] 
INFO[0164] [healthcheck] service [kube-scheduler] on host [192.168.100.4] is healthy 
INFO[0164] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0164] Starting container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0164] [controlplane] Successfully started [rke-log-linker] container on host [192.168.100.4] 
INFO[0164] Removing container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0164] [remove/rke-log-linker] Successfully removed container on host [192.168.100.4] 
INFO[0164] [controlplane] Successfully started Controller Plane.. 
INFO[0164] [authz] Creating rke-job-deployer ServiceAccount 
INFO[0164] [authz] rke-job-deployer ServiceAccount created successfully 
INFO[0164] [authz] Creating system:node ClusterRoleBinding 
INFO[0164] [authz] system:node ClusterRoleBinding created successfully 
INFO[0164] [authz] Creating kube-apiserver proxy ClusterRole and ClusterRoleBinding 
INFO[0164] [authz] kube-apiserver proxy ClusterRole and ClusterRoleBinding created successfully 
INFO[0164] Successfully Deployed state file at [./cluster.rkestate] 
INFO[0164] [state] Saving full cluster state to Kubernetes 
INFO[0164] [state] Successfully Saved full cluster state to Kubernetes ConfigMap: full-cluster-state 
INFO[0164] [worker] Building up Worker Plane..          
INFO[0164] Checking if container [service-sidekick] is running on host [192.168.100.4], try #1 
INFO[0164] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0164] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0164] [sidekick] Sidekick container already created on host [192.168.100.4] 
INFO[0164] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.4] 
INFO[0164] Starting container [kubelet] on host [192.168.100.4], try #1 
INFO[0164] Starting container [nginx-proxy] on host [192.168.100.6], try #1 
INFO[0164] Starting container [nginx-proxy] on host [192.168.100.5], try #1 
INFO[0164] [worker] Successfully started [kubelet] container on host [192.168.100.4] 
INFO[0164] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.100.4] 
INFO[0165] [worker] Successfully started [nginx-proxy] container on host [192.168.100.6] 
INFO[0165] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0165] [worker] Successfully started [nginx-proxy] container on host [192.168.100.5] 
INFO[0165] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0165] Starting container [rke-log-linker] on host [192.168.100.6], try #1 
INFO[0165] Starting container [rke-log-linker] on host [192.168.100.5], try #1 
INFO[0165] [worker] Successfully started [rke-log-linker] container on host [192.168.100.6] 
INFO[0165] Removing container [rke-log-linker] on host [192.168.100.6], try #1 
INFO[0165] [worker] Successfully started [rke-log-linker] container on host [192.168.100.5] 
INFO[0165] Removing container [rke-log-linker] on host [192.168.100.5], try #1 
INFO[0165] [remove/rke-log-linker] Successfully removed container on host [192.168.100.6] 
INFO[0165] Checking if container [service-sidekick] is running on host [192.168.100.6], try #1 
INFO[0165] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0165] [remove/rke-log-linker] Successfully removed container on host [192.168.100.5] 
INFO[0165] Checking if container [service-sidekick] is running on host [192.168.100.5], try #1 
INFO[0165] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0165] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.6] 
INFO[0165] Starting container [kubelet] on host [192.168.100.6], try #1 
INFO[0165] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.5] 
INFO[0165] Starting container [kubelet] on host [192.168.100.5], try #1 
INFO[0165] [worker] Successfully started [kubelet] container on host [192.168.100.6] 
INFO[0165] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.100.6] 
INFO[0165] [worker] Successfully started [kubelet] container on host [192.168.100.5] 
INFO[0165] [healthcheck] Start Healthcheck on service [kubelet] on host [192.168.100.5] 
INFO[0175] [healthcheck] service [kubelet] on host [192.168.100.4] is healthy 
INFO[0175] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0175] Starting container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0175] [worker] Successfully started [rke-log-linker] container on host [192.168.100.4] 
INFO[0175] Removing container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0175] [remove/rke-log-linker] Successfully removed container on host [192.168.100.4] 
INFO[0175] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.4] 
INFO[0175] Starting container [kube-proxy] on host [192.168.100.4], try #1 
INFO[0175] [worker] Successfully started [kube-proxy] container on host [192.168.100.4] 
INFO[0175] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.100.4] 
INFO[0175] [healthcheck] service [kubelet] on host [192.168.100.6] is healthy 
INFO[0175] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0175] Starting container [rke-log-linker] on host [192.168.100.6], try #1 
INFO[0175] [healthcheck] service [kubelet] on host [192.168.100.5] is healthy 
INFO[0175] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0175] Starting container [rke-log-linker] on host [192.168.100.5], try #1 
INFO[0176] [worker] Successfully started [rke-log-linker] container on host [192.168.100.6] 
INFO[0176] Removing container [rke-log-linker] on host [192.168.100.6], try #1 
INFO[0176] [worker] Successfully started [rke-log-linker] container on host [192.168.100.5] 
INFO[0176] Removing container [rke-log-linker] on host [192.168.100.5], try #1 
INFO[0176] [remove/rke-log-linker] Successfully removed container on host [192.168.100.6] 
INFO[0176] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.6] 
INFO[0176] Starting container [kube-proxy] on host [192.168.100.6], try #1 
INFO[0176] [worker] Successfully started [kube-proxy] container on host [192.168.100.6] 
INFO[0176] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.100.6] 
INFO[0176] [remove/rke-log-linker] Successfully removed container on host [192.168.100.5] 
INFO[0176] Image [rancher/hyperkube:v1.20.8-rancher1] exists on host [192.168.100.5] 
INFO[0176] Starting container [kube-proxy] on host [192.168.100.5], try #1 
INFO[0176] [healthcheck] service [kube-proxy] on host [192.168.100.6] is healthy 
INFO[0176] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0176] [worker] Successfully started [kube-proxy] container on host [192.168.100.5] 
INFO[0176] [healthcheck] Start Healthcheck on service [kube-proxy] on host [192.168.100.5] 
INFO[0176] Starting container [rke-log-linker] on host [192.168.100.6], try #1 
INFO[0176] [healthcheck] service [kube-proxy] on host [192.168.100.5] is healthy 
INFO[0176] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0176] Starting container [rke-log-linker] on host [192.168.100.5], try #1 
INFO[0176] [worker] Successfully started [rke-log-linker] container on host [192.168.100.6] 
INFO[0176] Removing container [rke-log-linker] on host [192.168.100.6], try #1 
INFO[0176] [worker] Successfully started [rke-log-linker] container on host [192.168.100.5] 
INFO[0176] Removing container [rke-log-linker] on host [192.168.100.5], try #1 
INFO[0176] [remove/rke-log-linker] Successfully removed container on host [192.168.100.6] 
INFO[0176] [remove/rke-log-linker] Successfully removed container on host [192.168.100.5] 
INFO[0180] [healthcheck] service [kube-proxy] on host [192.168.100.4] is healthy 
INFO[0180] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0181] Starting container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0181] [worker] Successfully started [rke-log-linker] container on host [192.168.100.4] 
INFO[0181] Removing container [rke-log-linker] on host [192.168.100.4], try #1 
INFO[0181] [remove/rke-log-linker] Successfully removed container on host [192.168.100.4] 
INFO[0181] [worker] Successfully started Worker Plane.. 
INFO[0181] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0181] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0181] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0181] Starting container [rke-log-cleaner] on host [192.168.100.5], try #1 
INFO[0181] Starting container [rke-log-cleaner] on host [192.168.100.6], try #1 
INFO[0181] Starting container [rke-log-cleaner] on host [192.168.100.4], try #1 
INFO[0181] [cleanup] Successfully started [rke-log-cleaner] container on host [192.168.100.4] 
INFO[0181] [cleanup] Successfully started [rke-log-cleaner] container on host [192.168.100.5] 
INFO[0181] Removing container [rke-log-cleaner] on host [192.168.100.4], try #1 
INFO[0181] [cleanup] Successfully started [rke-log-cleaner] container on host [192.168.100.6] 
INFO[0181] Removing container [rke-log-cleaner] on host [192.168.100.5], try #1 
INFO[0181] Removing container [rke-log-cleaner] on host [192.168.100.6], try #1 
INFO[0181] [remove/rke-log-cleaner] Successfully removed container on host [192.168.100.5] 
INFO[0181] [remove/rke-log-cleaner] Successfully removed container on host [192.168.100.4] 
INFO[0181] [remove/rke-log-cleaner] Successfully removed container on host [192.168.100.6] 
INFO[0181] [sync] Syncing nodes Labels and Taints       
INFO[0181] [sync] Successfully synced nodes Labels and Taints 
INFO[0181] [network] Setting up network plugin: flannel 
INFO[0181] [addons] Saving ConfigMap for addon rke-network-plugin to Kubernetes 
INFO[0181] [addons] Successfully saved ConfigMap for addon rke-network-plugin to Kubernetes 
INFO[0181] [addons] Executing deploy job rke-network-plugin 
INFO[0196] [addons] Setting up coredns                  
INFO[0196] [addons] Saving ConfigMap for addon rke-coredns-addon to Kubernetes 
INFO[0196] [addons] Successfully saved ConfigMap for addon rke-coredns-addon to Kubernetes 
INFO[0196] [addons] Executing deploy job rke-coredns-addon 
INFO[0201] [addons] CoreDNS deployed successfully       
INFO[0201] [dns] DNS provider coredns deployed successfully 
INFO[0201] [addons] Setting up Metrics Server           
INFO[0201] [addons] Saving ConfigMap for addon rke-metrics-addon to Kubernetes 
INFO[0201] [addons] Successfully saved ConfigMap for addon rke-metrics-addon to Kubernetes 
INFO[0201] [addons] Executing deploy job rke-metrics-addon 
INFO[0207] [addons] Metrics Server deployed successfully 
INFO[0207] [ingress] Setting up nginx ingress controller 
INFO[0207] [addons] Saving ConfigMap for addon rke-ingress-controller to Kubernetes 
INFO[0207] [addons] Successfully saved ConfigMap for addon rke-ingress-controller to Kubernetes 
INFO[0207] [addons] Executing deploy job rke-ingress-controller 
INFO[0212] [ingress] ingress controller nginx deployed successfully 
INFO[0212] [addons] Setting up user addons              
INFO[0212] [addons] no user addons defined              
INFO[0212] Finished building Kubernetes cluster successfully 
```

:::

`Finished building Kubernetes cluster successfully` の出力がされて、無事にクラスタ構築が完了しました。(Docker がインストールされていなかったり、SSH 接続できなかったりした場合は、インストール中にエラーが発生します。)

すると `cluster.rkestate` と `kube_config_cluster.yml` が作成されます。

```bash
$ ls
cluster.rkestate         cluster.yml              kube_config_cluster.yml
```

この状態で `kubectl` を実行するときは、`--kubeconfig=kube_config_cluster.yml` オプションを付けます。

```bash
kubectl --kubeconfig=kube_config_cluster.yml get all -A
```

はじめに作成されたものは以下になっていました。

:::details 実行ログはこちらです。

```bash
$ kubectl --kubeconfig=kube_config_cluster.yml get all -A
NAMESPACE	NAME                                          READY   STATUS      RESTARTS   AGE
ingress-nginx   pod/default-http-backend-6977475d9b-pb7x8     1/1     Running     0          2m29s
ingress-nginx   pod/nginx-ingress-controller-5z85v            1/1     Running     0          2m29s
ingress-nginx   pod/nginx-ingress-controller-8lbjn            1/1     Running     0          2m29s
ingress-nginx   pod/nginx-ingress-controller-zbwfp            1/1     Running     0          2m29s
kube-system     pod/coredns-55b58f978-6vb9b                   1/1     Running     0          2m38s
kube-system     pod/coredns-55b58f978-dwxz2                   1/1     Running     0          78s
kube-system     pod/coredns-autoscaler-76f8869cc9-mfw6v       1/1     Running     0          2m38s
kube-system     pod/kube-flannel-bfnvj                        2/2     Running     0          2m44s
kube-system     pod/kube-flannel-mp59l                        2/2     Running     0          2m44s
kube-system     pod/kube-flannel-tcn6j                        2/2     Running     0          2m44s
kube-system     pod/metrics-server-55fdd84cd4-bdcrd           1/1     Running     0          2m33s
kube-system     pod/rke-coredns-addon-deploy-job-c7dmx        0/1     Completed   0          2m39s
kube-system     pod/rke-ingress-controller-deploy-job-kgrr2   0/1     Completed   0          2m29s
kube-system     pod/rke-metrics-addon-deploy-job-fhjnw        0/1     Completed   0          2m34s
kube-system     pod/rke-network-plugin-deploy-job-bxlxl       0/1     Completed   0          2m56s

NAMESPACE       NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default         service/kubernetes             ClusterIP   10.43.0.1       <none>        443/TCP                  3m29s
ingress-nginx   service/default-http-backend   ClusterIP   10.43.138.245   <none>        80/TCP                   2m29s
kube-system     service/kube-dns               ClusterIP   10.43.0.10      <none>        53/UDP,53/TCP,9153/TCP   2m38s
kube-system     service/metrics-server         ClusterIP   10.43.96.78     <none>        443/TCP                  2m33s

NAMESPACE       NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ingress-nginx   daemonset.apps/nginx-ingress-controller   3         3         3       3            3           <none>          2m29s
kube-system     daemonset.apps/kube-flannel               3         3         3       3            3           <none>          2m44s

NAMESPACE       NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
ingress-nginx   deployment.apps/default-http-backend   1/1     1            1           2m29s
kube-system     deployment.apps/coredns                2/2     2            2           2m38s
kube-system     deployment.apps/coredns-autoscaler     1/1     1            1           2m38s
kube-system     deployment.apps/metrics-server         1/1     1            1           2m33s

NAMESPACE       NAME                                              DESIRED   CURRENT   READY   AGE
ingress-nginx   replicaset.apps/default-http-backend-6977475d9b   1         1         1       2m29s
kube-system     replicaset.apps/coredns-55b58f978                 2         2         2       2m38s
kube-system     replicaset.apps/coredns-autoscaler-76f8869cc9     1         1         1       2m38s
kube-system     replicaset.apps/metrics-server-55fdd84cd4         1         1         1       2m33s

NAMESPACE     NAME                                          COMPLETIONS   DURATION   AGE
kube-system   job.batch/rke-coredns-addon-deploy-job        1/1           2s         2m39s
kube-system   job.batch/rke-ingress-controller-deploy-job   1/1           1s         2m29s
kube-system   job.batch/rke-metrics-addon-deploy-job        1/1           1s         2m34s
kube-system   job.batch/rke-network-plugin-deploy-job       1/1           12s        2m56s
```

:::

また、環境変数 `KUBECONFIG` に `kube_config_cluster.yml` のパスを指定することで、上述のオプションなしで実行できます。

```bash
export KUBECONFIG=$PWD/kube_config_cluster.yml
```

クラスタを削除するときは以下を実行するだけです。

```bash
rke remove
```

:::details 実行ログはこちらです。

```bash
$ rke remove
INFO[0000] Running RKE version: v1.2.9                  
Are you sure you want to remove Kubernetes cluster [y/n]: y
INFO[0003] Tearing down Kubernetes cluster              
INFO[0003] [dialer] Setup tunnel for host [192.168.100.6] 
INFO[0003] [dialer] Setup tunnel for host [192.168.100.5] 
INFO[0003] [dialer] Setup tunnel for host [192.168.100.4] 
INFO[0004] [worker] Tearing down Worker Plane..         
INFO[0004] Removing container [kubelet] on host [192.168.100.5], try #1 
INFO[0004] Removing container [kubelet] on host [192.168.100.4], try #1 
INFO[0004] Removing container [kubelet] on host [192.168.100.6], try #1 
INFO[0004] [remove/kubelet] Successfully removed container on host [192.168.100.5] 
INFO[0004] [remove/kubelet] Successfully removed container on host [192.168.100.6] 
INFO[0004] Removing container [kube-proxy] on host [192.168.100.5], try #1 
INFO[0004] Removing container [kube-proxy] on host [192.168.100.6], try #1 
INFO[0004] [remove/kubelet] Successfully removed container on host [192.168.100.4] 
INFO[0004] Removing container [kube-proxy] on host [192.168.100.4], try #1 
INFO[0004] [remove/kube-proxy] Successfully removed container on host [192.168.100.6] 
INFO[0004] Removing container [nginx-proxy] on host [192.168.100.6], try #1 
INFO[0004] [remove/kube-proxy] Successfully removed container on host [192.168.100.5] 
INFO[0004] Removing container [nginx-proxy] on host [192.168.100.5], try #1 
INFO[0004] [remove/kube-proxy] Successfully removed container on host [192.168.100.4] 
INFO[0004] Removing container [service-sidekick] on host [192.168.100.4], try #1 
INFO[0004] [remove/service-sidekick] Successfully removed container on host [192.168.100.4] 
INFO[0004] [remove/nginx-proxy] Successfully removed container on host [192.168.100.6] 
INFO[0004] Removing container [service-sidekick] on host [192.168.100.6], try #1 
INFO[0004] [remove/service-sidekick] Successfully removed container on host [192.168.100.6] 
INFO[0004] [remove/nginx-proxy] Successfully removed container on host [192.168.100.5] 
INFO[0004] Removing container [service-sidekick] on host [192.168.100.5], try #1 
INFO[0004] [remove/service-sidekick] Successfully removed container on host [192.168.100.5] 
INFO[0004] [worker] Successfully tore down Worker Plane.. 
INFO[0004] [controlplane] Tearing down the Controller Plane.. 
INFO[0004] Removing container [kube-apiserver] on host [192.168.100.4], try #1 
INFO[0004] [remove/kube-apiserver] Successfully removed container on host [192.168.100.4] 
INFO[0004] Removing container [kube-controller-manager] on host [192.168.100.4], try #1 
INFO[0004] [remove/kube-controller-manager] Successfully removed container on host [192.168.100.4] 
INFO[0004] Removing container [kube-scheduler] on host [192.168.100.4], try #1 
INFO[0004] [remove/kube-scheduler] Successfully removed container on host [192.168.100.4] 
INFO[0004] [controlplane] Successfully tore down Controller Plane.. 
INFO[0004] [etcd] Tearing down etcd plane..             
INFO[0004] Removing container [etcd] on host [192.168.100.4], try #1 
INFO[0004] [remove/etcd] Successfully removed container on host [192.168.100.4] 
INFO[0004] Removing container [etcd-rolling-snapshots] on host [192.168.100.4], try #1 
INFO[0004] [remove/etcd-rolling-snapshots] Successfully removed container on host [192.168.100.4] 
INFO[0004] [etcd] Successfully tore down etcd plane..   
INFO[0004] [hosts] Cleaning up host [192.168.100.4]     
INFO[0004] [hosts] Cleaning up host [192.168.100.4]     
INFO[0004] [hosts] Running cleaner container on host [192.168.100.4] 
INFO[0004] [hosts] Cleaning up host [192.168.100.6]     
INFO[0004] [hosts] Cleaning up host [192.168.100.6]     
INFO[0004] [hosts] Cleaning up host [192.168.100.5]     
INFO[0004] [hosts] Cleaning up host [192.168.100.5]     
INFO[0004] [hosts] Running cleaner container on host [192.168.100.5] 
INFO[0004] [hosts] Running cleaner container on host [192.168.100.6] 
INFO[0004] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0004] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0004] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0004] Starting container [kube-cleaner] on host [192.168.100.4], try #1 
INFO[0004] Starting container [kube-cleaner] on host [192.168.100.6], try #1 
INFO[0004] Starting container [kube-cleaner] on host [192.168.100.5], try #1 
INFO[0004] [kube-cleaner] Successfully started [kube-cleaner] container on host [192.168.100.4] 
INFO[0004] Waiting for [kube-cleaner] container to exit on host [192.168.100.4] 
INFO[0004] Container [kube-cleaner] is still running on host [192.168.100.4]: stderr: [find: /etc/kubernetes/.tmp: Resource busy
], stdout: [] 
INFO[0004] [kube-cleaner] Successfully started [kube-cleaner] container on host [192.168.100.6] 
INFO[0004] Waiting for [kube-cleaner] container to exit on host [192.168.100.6] 
INFO[0004] Container [kube-cleaner] is still running on host [192.168.100.6]: stderr: [find: /etc/kubernetes/.tmp: Resource busy
], stdout: [] 
INFO[0004] [kube-cleaner] Successfully started [kube-cleaner] container on host [192.168.100.5] 
INFO[0004] Waiting for [kube-cleaner] container to exit on host [192.168.100.5] 
INFO[0004] Container [kube-cleaner] is still running on host [192.168.100.5]: stderr: [find: /etc/kubernetes/.tmp: Resource busy
], stdout: [] 
INFO[0005] Waiting for [kube-cleaner] container to exit on host [192.168.100.4] 
INFO[0005] [hosts] Removing cleaner container on host [192.168.100.4] 
INFO[0005] Removing container [kube-cleaner] on host [192.168.100.4], try #1 
INFO[0005] [hosts] Removing dead container logs on host [192.168.100.4] 
INFO[0005] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.4] 
INFO[0005] Waiting for [kube-cleaner] container to exit on host [192.168.100.6] 
INFO[0005] [hosts] Removing cleaner container on host [192.168.100.6] 
INFO[0005] Removing container [kube-cleaner] on host [192.168.100.6], try #1 
INFO[0005] [hosts] Removing dead container logs on host [192.168.100.6] 
INFO[0005] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.6] 
INFO[0005] Waiting for [kube-cleaner] container to exit on host [192.168.100.5] 
INFO[0005] [hosts] Removing cleaner container on host [192.168.100.5] 
INFO[0005] Removing container [kube-cleaner] on host [192.168.100.5], try #1 
INFO[0005] [hosts] Removing dead container logs on host [192.168.100.5] 
INFO[0006] Image [rancher/rke-tools:v0.1.75] exists on host [192.168.100.5] 
INFO[0006] Starting container [rke-log-cleaner] on host [192.168.100.4], try #1 
INFO[0006] Starting container [rke-log-cleaner] on host [192.168.100.6], try #1 
INFO[0006] Starting container [rke-log-cleaner] on host [192.168.100.5], try #1 
INFO[0006] [cleanup] Successfully started [rke-log-cleaner] container on host [192.168.100.4] 
INFO[0006] Removing container [rke-log-cleaner] on host [192.168.100.4], try #1 
INFO[0006] [cleanup] Successfully started [rke-log-cleaner] container on host [192.168.100.5] 
INFO[0006] Removing container [rke-log-cleaner] on host [192.168.100.5], try #1 
INFO[0006] [cleanup] Successfully started [rke-log-cleaner] container on host [192.168.100.6] 
INFO[0006] Removing container [rke-log-cleaner] on host [192.168.100.6], try #1 
INFO[0006] [remove/rke-log-cleaner] Successfully removed container on host [192.168.100.4] 
INFO[0006] [hosts] Successfully cleaned up host [192.168.100.4] 
INFO[0006] [remove/rke-log-cleaner] Successfully removed container on host [192.168.100.5] 
INFO[0006] [hosts] Successfully cleaned up host [192.168.100.5] 
INFO[0006] [remove/rke-log-cleaner] Successfully removed container on host [192.168.100.6] 
INFO[0006] [hosts] Successfully cleaned up host [192.168.100.6] 
INFO[0006] Removing local admin Kubeconfig: ./kube_config_cluster.yml 
INFO[0006] Local admin Kubeconfig removed successfully  
INFO[0006] Removing state file: ./cluster.rkestate      
INFO[0006] State file removed successfully              
INFO[0006] Cluster removed successfully
```

:::

# Terraform RKE providerを使用してみる

https://registry.terraform.io/providers/rancher/rke/latest

https://github.com/rancher/terraform-provider-rke

Terraform rke provider を使用すると、HCL でクラスタを定義できます。
Terraform との連携により、クラスタとなるインスタンスの作成からクラスタ構築までを一元的に管理できます。(terraform variables をそのまま使える)

先ほど作ったものと同様の内容のものを tf ファイルで定義したものは以下になります。(各 node の前提条件は満たしているものとします)

```hcl
resource "rke_cluster" "cluster" {
  nodes {
    address          = esxi_guest.k8s-master01.ip_address
    internal_address = "192.168.101.4"
    user             = "kube"
    ssh_key          = file("~/.ssh/id_rsa.kube")
    role             = ["controlplane", "worker", "etcd"]
  }
  nodes {
    address          = esxi_guest.k8s-worker01.ip_address
    internal_address = "192.168.101.5"
    user             = "kube"
    ssh_key          = file("~/.ssh/id_rsa.kube")
    role             = ["worker"]
  }
  nodes {
    address          = esxi_guest.k8s-worker02.ip_address
    internal_address = "192.168.101.6"
    user             = "kube"
    ssh_key          = file("~/.ssh/id_rsa.kube")
    role             = ["worker"]
  }
  network {
    plugin = "flannel"
  }

  delay_on_creation = 10
}

resource "local_file" "kube_cluster_yaml" {
  filename = "kube_config_cluster.yml"
  content  = rke_cluster.cluster.kube_config_yaml
}

resource "local_file" "rke_cluster_yaml" {
  filename = "cluster.yml"
  content  = rke_cluster.cluster.rke_cluster_yaml
}

resource "local_file" "rke_state" {
  filename = "cluster.rkestate"
  content  = rke_cluster.cluster.rke_state
}
```

この状態で `terraform apply` すると、CLI で実施した `rke up` と同様のことを実行できます。(実行環境は CLI 同様に Mac です)

また、 `cluster.yml` から `rke up` する時と比べて少しだけ不安定な要素があったので、以下のオプションを追加しています。

- `delay_on_creation = 10`
  - クラスタ作成 10 秒を遅らせているオプションです。
  - rke リソースは、クラスタとなるインスタンス起動後に実行されます。
  - インスタンス側でエラーがあった場合、そのインスタンスはクラスタに追加されないまま終了してしまいます(sshd が起動されていないなど)
  - そのため、インスタンスに必要なプロセスを十分待った上で rke が実行されるようにしました。
- `resource.local_file`:  以下の `rke up` した時に作成されるファイルをローカルに保存する設定です。
  - `kube_cluster_yaml`
    - terraform 実行ホストから、 `kubectl` コマンドが実行できるようにします。
    - CLI の時はデフォルトで作成されてましたが、terraform はこのように指定する必要があります。
  - `rke_cluster_yaml`
    - CLI 用の yml もバックアップとして一応作成されるようにしています。
  - `rke_state`
    - クラスタ削除する時、デフォルトだと `terraform destroy` しただけでは削除されないようです。
    - その為、`rke remove` コマンドが実行できるようにこのファイルを保存しています。
    - CLI の時はデフォルトで作成されてましたが、terraform はこのように指定する必要があります。

# Reference

https://febc-yamamoto.hatenablog.jp/entry/introduce-terraform-provider-rke

https://recruit.gmo.jp/engineer/jisedai/blog/rancher_kubernetes_engine/

https://tech.virtualtech.jp/entry/2019/09/03/142114
