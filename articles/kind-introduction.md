---
title: "Macにkindをインストールする"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kind", "kubernetes", "mac"]
published: true
---

お手軽に Kubernetes クラスタが構築できる kind を試してみます。

# Get Started

https://kind.sigs.k8s.io/docs/user/quick-start/

Mac に kind をインストールします。

```bash
brew install kind
```

次にクラスタ作成用の `kind.yaml` を作成します。

```yaml:kind.yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
    image: kindest/node:v1.18.2
  - role: control-plane
    image: kindest/node:v1.18.2
  - role: control-plane
    image: kindest/node:v1.18.2
  - role: worker
    image: kindest/node:v1.18.2
  - role: worker
    image: kindest/node:v1.18.2
  - role: worker
    image: kindest/node:v1.18.2
```

kind コマンドでクラスタ作成します。(`—-name` を指定しない場合はデフォルトで kind という名前になります。)

```bash
kind create cluster --config kind.yaml --name kindcluster
```

:::details 実行ログはこちらです。

```bash
$ kind create cluster --config kind.yaml --name kindcluster
Creating cluster "kindcluster" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼 
 ✓ Preparing nodes 📦 📦 📦 📦 📦 📦  
 ✓ Configuring the external load balancer ⚖️ 
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✗ Joining more control-plane nodes 🎮 
ERROR: failed to create cluster: failed to join node with kubeadm: command "docker exec --privileged kindcluster-control-plane2 kubeadm join --config /kind/kubeadm.conf --skip-phases=preflight --v=6" failed with error: exit status 1
Command Output: W0618 04:40:57.597369    1132 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
I0618 04:40:57.597538    1132 join.go:371] [preflight] found NodeName empty; using OS hostname as NodeName
I0618 04:40:57.597577    1132 joinconfiguration.go:74] loading configuration from "/kind/kubeadm.conf"
I0618 04:40:57.612779    1132 controlplaneprepare.go:211] [download-certs] Skipping certs download
I0618 04:40:57.614907    1132 join.go:441] [preflight] Discovering cluster-info
I0618 04:40:57.617791    1132 token.go:78] [discovery] Created cluster-info discovery client, requesting info from "kindcluster-external-load-balancer:6443"
I0618 04:40:57.743832    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 111 milliseconds
I0618 04:40:57.749234    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:40:57.777647    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 24 milliseconds
I0618 04:40:57.783772    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:41:03.457227    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 14 milliseconds
I0618 04:41:03.459118    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:41:09.105840    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 8 milliseconds
I0618 04:41:09.107440    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:41:15.138724    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 30 milliseconds
I0618 04:41:15.147451    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:41:20.261346    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 9 milliseconds
I0618 04:41:20.263159    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:41:25.507058    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 7 milliseconds
I0618 04:41:25.513369    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:41:31.913585    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 1217 milliseconds
I0618 04:41:31.928891    1132 token.go:221] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "abcdef", will try again
I0618 04:41:37.431273    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s 200 OK in 45 milliseconds
I0618 04:41:37.532249    1132 token.go:103] [discovery] Cluster info signature and contents are valid and no TLS pinning was specified, will use API Server "kindcluster-external-load-balancer:6443"
I0618 04:41:37.534776    1132 discovery.go:51] [discovery] Using provided TLSBootstrapToken as authentication credentials for the join process
I0618 04:41:37.536998    1132 join.go:455] [preflight] Fetching init configuration
I0618 04:41:37.540736    1132 join.go:493] [preflight] Retrieving KubeConfig objects
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
I0618 04:41:37.734724    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config?timeout=10s 200 OK in 148 milliseconds
I0618 04:41:37.828077    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-system/configmaps/kube-proxy?timeout=10s 200 OK in 78 milliseconds
I0618 04:41:37.868066    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-system/configmaps/kubelet-config-1.18?timeout=10s 200 OK in 21 milliseconds
I0618 04:41:37.929239    1132 interface.go:400] Looking for default routes with IPv4 addresses
I0618 04:41:37.930200    1132 interface.go:405] Default route transits interface "eth0"
I0618 04:41:37.931808    1132 interface.go:208] Interface eth0 is up
I0618 04:41:37.932295    1132 interface.go:256] Interface "eth0" has 3 addresses :[172.21.0.4/16 fc00:f853:ccd:e793::4/64 fe80::42:acff:fe15:4/64].
I0618 04:41:37.936357    1132 interface.go:223] Checking addr  172.21.0.4/16.
I0618 04:41:37.937279    1132 interface.go:230] IP found 172.21.0.4
I0618 04:41:37.938143    1132 interface.go:262] Found valid IPv4 address 172.21.0.4 for interface "eth0".
I0618 04:41:37.938998    1132 interface.go:411] Found active IP 172.21.0.4 
[certs] Using certificateDir folder "/etc/kubernetes/pki"
I0618 04:41:37.952288    1132 certs.go:38] creating PKI assets
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kindcluster-control-plane2 localhost] and IPs [172.21.0.4 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kindcluster-control-plane2 localhost] and IPs [172.21.0.4 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kindcluster-control-plane2 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local kindcluster-external-load-balancer localhost] and IPs [10.96.0.1 172.21.0.4 127.0.0.1]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
I0618 04:41:49.934856    1132 certs.go:69] creating new public/private key files for signing service account users
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/admin.conf"
I0618 04:41:51.732454    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/admin.conf
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
I0618 04:41:53.736835    1132 manifests.go:91] [control-plane] getting StaticPodSpecs
W0618 04:41:53.748868    1132 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
I0618 04:41:53.759129    1132 manifests.go:104] [control-plane] adding volume "ca-certs" for component "kube-apiserver"
I0618 04:41:53.760245    1132 manifests.go:104] [control-plane] adding volume "etc-ca-certificates" for component "kube-apiserver"
I0618 04:41:53.761188    1132 manifests.go:104] [control-plane] adding volume "k8s-certs" for component "kube-apiserver"
I0618 04:41:53.761946    1132 manifests.go:104] [control-plane] adding volume "usr-local-share-ca-certificates" for component "kube-apiserver"
I0618 04:41:53.762561    1132 manifests.go:104] [control-plane] adding volume "usr-share-ca-certificates" for component "kube-apiserver"
I0618 04:41:53.851316    1132 manifests.go:121] [control-plane] wrote static Pod manifest for component "kube-apiserver" to "/etc/kubernetes/manifests/kube-apiserver.yaml"
I0618 04:41:53.851418    1132 manifests.go:91] [control-plane] getting StaticPodSpecs
W0618 04:41:53.851724    1132 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
I0618 04:41:53.853747    1132 manifests.go:104] [control-plane] adding volume "ca-certs" for component "kube-controller-manager"
I0618 04:41:53.853828    1132 manifests.go:104] [control-plane] adding volume "etc-ca-certificates" for component "kube-controller-manager"
I0618 04:41:53.853883    1132 manifests.go:104] [control-plane] adding volume "flexvolume-dir" for component "kube-controller-manager"
I0618 04:41:53.853923    1132 manifests.go:104] [control-plane] adding volume "k8s-certs" for component "kube-controller-manager"
I0618 04:41:53.853963    1132 manifests.go:104] [control-plane] adding volume "kubeconfig" for component "kube-controller-manager"
I0618 04:41:53.855424    1132 manifests.go:104] [control-plane] adding volume "usr-local-share-ca-certificates" for component "kube-controller-manager"
I0618 04:41:53.855457    1132 manifests.go:104] [control-plane] adding volume "usr-share-ca-certificates" for component "kube-controller-manager"
I0618 04:41:53.859373    1132 manifests.go:121] [control-plane] wrote static Pod manifest for component "kube-controller-manager" to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
I0618 04:41:53.859457    1132 manifests.go:91] [control-plane] getting StaticPodSpecs
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0618 04:41:53.860117    1132 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
I0618 04:41:53.862013    1132 manifests.go:104] [control-plane] adding volume "kubeconfig" for component "kube-scheduler"
I0618 04:41:53.866646    1132 manifests.go:121] [control-plane] wrote static Pod manifest for component "kube-scheduler" to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[check-etcd] Checking that the etcd cluster is healthy
I0618 04:41:53.869265    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/admin.conf
I0618 04:41:53.884929    1132 local.go:78] [etcd] Checking etcd cluster health
I0618 04:41:53.885025    1132 local.go:81] creating etcd client that connects to etcd pods
I0618 04:41:53.887942    1132 etcd.go:178] retrieving etcd endpoints from "kubeadm.kubernetes.io/etcd.advertise-client-urls" annotation in etcd Pods
I0618 04:41:54.113096    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-system/pods?labelSelector=component%3Detcd%2Ctier%3Dcontrol-plane 200 OK in 220 milliseconds
I0618 04:41:54.144454    1132 etcd.go:102] etcd endpoints read from pods: https://172.21.0.8:2379
I0618 04:41:54.336473    1132 etcd.go:250] etcd endpoints read from etcd: https://172.21.0.8:2379
I0618 04:41:54.338964    1132 etcd.go:120] update etcd endpoints: https://172.21.0.8:2379
I0618 04:41:55.053630    1132 kubelet.go:111] [kubelet-start] writing bootstrap kubelet config file at /etc/kubernetes/bootstrap-kubelet.conf
I0618 04:41:55.067800    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/bootstrap-kubelet.conf
I0618 04:41:55.070284    1132 kubelet.go:145] [kubelet-start] Checking for an existing Node in the cluster with name "kindcluster-control-plane2" and status "Ready"
I0618 04:41:55.475797    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 385 milliseconds
I0618 04:41:55.521985    1132 kubelet.go:159] [kubelet-start] Stopping the kubelet
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
I0618 04:41:55.758149    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/namespaces/kube-system/configmaps/kubelet-config-1.18?timeout=10s 200 OK in 70 milliseconds
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
I0618 04:42:00.200716    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
I0618 04:42:00.586912    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
I0618 04:42:01.087936    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
I0618 04:42:01.587014    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
I0618 04:42:02.086447    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
I0618 04:42:02.586378    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
I0618 04:42:02.821771    1132 cert_rotation.go:137] Starting client certificate rotation controller
I0618 04:42:02.967847    1132 loader.go:375] Config loaded from file:  /etc/kubernetes/kubelet.conf
I0618 04:42:02.992423    1132 kubelet.go:194] [kubelet-start] preserving the crisocket information for the node
I0618 04:42:03.010205    1132 patchnode.go:30] [patchnode] Uploading the CRI Socket information "unix:///run/containerd/containerd.sock" to the Node API object "kindcluster-control-plane2" as an annotation
I0618 04:42:04.513167    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 988 milliseconds
I0618 04:42:05.122317    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 105 milliseconds
I0618 04:42:05.582423    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 68 milliseconds
I0618 04:42:06.307800    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 295 milliseconds
I0618 04:42:06.791306    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 267 milliseconds
I0618 04:42:07.145071    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 131 milliseconds
I0618 04:42:07.775408    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 250 milliseconds
I0618 04:42:09.146948    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 633 milliseconds
I0618 04:42:09.538758    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 25 milliseconds
I0618 04:42:10.083837    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 70 milliseconds
I0618 04:42:10.792476    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 404 Not Found in 278 milliseconds
I0618 04:42:12.119052    1132 round_trippers.go:443] GET https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s 200 OK in 1130 milliseconds
I0618 04:42:22.428329    1132 round_trippers.go:443] PATCH https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s  in 10076 milliseconds
Patch https://kindcluster-external-load-balancer:6443/api/v1/nodes/kindcluster-control-plane2?timeout=10s: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
error patching node "kindcluster-control-plane2" through apiserver
k8s.io/kubernetes/cmd/kubeadm/app/util/apiclient.PatchNodeOnce.func1
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/util/apiclient/idempotency.go:305
k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.runConditionWithCrashProtection
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:211
k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.WaitFor
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:528
k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.pollInternal
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:414
k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.Poll
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:408
k8s.io/kubernetes/cmd/kubeadm/app/util/apiclient.PatchNode
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/util/apiclient/idempotency.go:318
k8s.io/kubernetes/cmd/kubeadm/app/phases/patchnode.AnnotateCRISocket
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/phases/patchnode/patchnode.go:32
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/join.runKubeletStartJoinPhase
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/join/kubelet.go:195
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run.func1
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:234
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).visitAll
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:422
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:207
k8s.io/kubernetes/cmd/kubeadm/app/cmd.NewCmdJoin.func1
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/join.go:170
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).execute
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:826
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).ExecuteC
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:914
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).Execute
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:864
k8s.io/kubernetes/cmd/kubeadm/app.Run
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/kubeadm.go:50
main.main
	_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/kubeadm.go:25
runtime.main
	/usr/local/go/src/runtime/proc.go:203
runtime.goexit
	/usr/local/go/src/runtime/asm_amd64.s:1357
error uploading crisocket
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/join.runKubeletStartJoinPhase
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/join/kubelet.go:196
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run.func1
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:234
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).visitAll
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:422
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:207
k8s.io/kubernetes/cmd/kubeadm/app/cmd.NewCmdJoin.func1
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/join.go:170
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).execute
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:826
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).ExecuteC
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:914
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).Execute
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:864
k8s.io/kubernetes/cmd/kubeadm/app.Run
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/kubeadm.go:50
main.main
	_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/kubeadm.go:25
runtime.main
	/usr/local/go/src/runtime/proc.go:203
runtime.goexit
	/usr/local/go/src/runtime/asm_amd64.s:1357
error execution phase kubelet-start
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run.func1
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:235
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).visitAll
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:422
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:207
k8s.io/kubernetes/cmd/kubeadm/app/cmd.NewCmdJoin.func1
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/cmd/join.go:170
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).execute
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:826
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).ExecuteC
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:914
k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).Execute
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:864
k8s.io/kubernetes/cmd/kubeadm/app.Run
	/go/src/k8s.io/kubernetes/_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/app/kubeadm.go:50
main.main
	_output/dockerized/go/src/k8s.io/kubernetes/cmd/kubeadm/kubeadm.go:25
runtime.main
	/usr/local/go/src/runtime/proc.go:203
runtime.goexit
	/usr/local/go/src/runtime/asm_amd64.s:1357
```

:::

Timeout なのでおそらくリソース不足のエラーです。ちなみにエラーで終了した場合、コンテナは削除されていました。

```bash
$ docker container ls
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
$ docker container ls -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

使用するリソースをデフォルトの設定から以下に変更して試してみました。

- CPU: 2
- Memory: 1.00GB → 2.00GB
- Swap: 1GB
- Disk image size: 59.6

![docker-for-mac-resources.png](https://firebasestorage.googleapis.com/v0/b/prod-ticket-app.appspot.com/o/post%2Fyumenomatayume%2F1626023137341?alt=media&token=464fa44c-5450-4b35-8e91-74466aae4e3a)

スペックを修正しましたので、リトライします。

```bash
$ kind create cluster --config kind.yaml --name kindcluster
Creating cluster "kindcluster" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼 
 ✓ Preparing nodes 📦 📦 📦 📦 📦 📦  
 ✓ Configuring the external load balancer ⚖️ 
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining more control-plane nodes 🎮 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-kindcluster"
You can now use your cluster with:

kubectl cluster-info --context kind-kindcluster

Thanks for using kind! 😊
```

無事に起動しました。 🎉🎉🎉

context を kind-kindcluster に切り替えます。

```bash
$ kubectl config use-context kind-kindcluster
Switched to context "kind-kindcluster".
```

Kubernetes Nodes が構築されている事が確認できました。

```bash
$ kubectl get nodes
NAME                         STATUS   ROLES    AGE   VERSION
kindcluster-control-plane    Ready    master   15m   v1.18.2
kindcluster-control-plane2   Ready    master   13m   v1.18.2
kindcluster-control-plane3   Ready    master   11m   v1.18.2
kindcluster-worker           Ready    <none>   10m   v1.18.2
kindcluster-worker2          Ready    <none>   10m   v1.18.2
kindcluster-worker3          Ready    <none>   10m   v1.18.2
```

# 終わりに

kind の公式サイトの「Settings for Docker Desktop」に必要な推奨スペックの記載がありました。

- vCPU: 4
- Memory: 6-8GB

多くのリソースを必要とするので、Kubernetes 検証する際にはリソースに余裕を持たせておき必要がありそうです。

# Reference

https://kind.sigs.k8s.io/docs/user/quick-start/
