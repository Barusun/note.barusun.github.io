[TOC]

# kubernetes 集群部署指南（kubeadm）

| Hostname          | IPAddress      | MacAddress        |
| ----------------- | -------------- | ----------------- |
| Kubernetes-Master | 192.168.17.130 | 00:0c:29:6f:01:1a |
| Kubernetes-Node1  | 192.168.17.131 | 00:0c:29:44:a7:c9 |
| Kubernetes-Node2  | 192.168.17.132 | 00:0c:29:b5:c5:f4 |

| pod-network-cidr | service-cidr |
| ---------------- | ------------ |
| 10.244.0.0/16    | 10.96.0.0/16 |



## 安装runtime

>容器引擎有好多种，都是掉用的kubernetes的CRI（Container Runtime Interface）
>
>Docker（默认）
>
>cri-o
>
>f'rakti
>
>containerd

### 安装Docker

>执行设备：Master、Node

~~~
apt update
apt-get -y install docker-ce=18.06.1~ce~3-0~ubuntu
systemctl enable docker
systemctl restart docker
systemctl status docker

~~~

### 安装Kubeadm、kubectl、kubelet

> 执行设备：Master、Node

1. 首先添加apt-key（这个证书的问题需要注意）

   ~~~
   apt update && sudo apt install -y apt-transport-https curl
   curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
   ~~~

2. 添加kubernetes源

   ~~~
   cat >/etc/apt/sources.list.d/kubernetes.list <<EOF
   deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
   EOF
   ~~~

3. 安装

   ~~~
   apt update
   apt install -y kubelet kubeadm kubectl
   apt-mark hold kubelet kubeadm kubectl
   ~~~

### 初始化Master

~~~
kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.13.1 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/16
输出：
[init] Using Kubernetes version: v1.13.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.17.130]
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubernetes-master localhost] and IPs [192.168.17.130 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubernetes-master localhost] and IPs [192.168.17.130 127.0.0.1 ::1]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 26.005496 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kubernetes-master" as an annotation
[mark-control-plane] Marking the node kubernetes-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubernetes-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 671p99.pmvkuvnxic6dcghr
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.17.130:6443 --token 671p99.pmvkuvnxic6dcghr --discovery-token-ca-cert-hash sha256:6f6840323044e48fc21c965695a3489f75046e3f7a4b0a37da47349b33a09fce
~~~

> Node:
>
> 需要记录：
>
> ~~~
> kubeadm join 192.168.17.130:6443 --token 671p99.pmvkuvnxic6dcghr --discovery-token-ca-cert-hash sha256:6f6840323044e48fc21c965695a3489f75046e3f7a4b0a37da47349b33a09fce
> ~~~
>
> 以上信息为kubernetes节点加入集群使用的命令

管理集群

~~~
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
~~~

查看pod状态

~~~shell
root@Kubernetes-Master:~# kubectl get po --all-namespaces -o wide
NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
kube-system   coredns-78d4cf999f-gqq8g                    0/1     Pending   0          3m31s   <none>           <none>              <none>           <none>
kube-system   coredns-78d4cf999f-p76dh                    0/1     Pending   0          3m31s   <none>           <none>              <none>           <none>
kube-system   etcd-kubernetes-master                      1/1     Running   0          2m35s   192.168.17.130   kubernetes-master   <none>           <none>
kube-system   kube-apiserver-kubernetes-master            1/1     Running   0          2m46s   192.168.17.130   kubernetes-master   <none>           <none>
kube-system   kube-controller-manager-kubernetes-master   1/1     Running   0          2m57s   192.168.17.130   kubernetes-master   <none>           <none>
kube-system   kube-proxy-5hsrw                            1/1     Running   0          3m31s   192.168.17.130   kubernetes-master   <none>           <none>
kube-system   kube-scheduler-kubernetes-master            1/1     Running   0          2m33s   192.168.17.130   kubernetes-master   <none>           <none>
~~~



### 安装网络插件

1. 创建文件夹

```shell
mkdir -p /data/kubernetes/flannel
```

2. 下载配置文件

~~~
wget https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
~~~

3. 使用配置文件

~~~
cd /data/kubernetes/flannel
kubectl apply -f kube-flannel.yml
输出：
root@Kubernetes-Master:/data/kubernetes/flannel# kubectl apply -f kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
~~~

> 备注：个文件里面搜flanneld有五个不同架构下的DaemonSet配置, 需要在amd64架构那个DaemonSet的args下增加--iface <node-ip-iface-name>（只有一张网卡的时候，加不加都可以：flannel和calico会从每张网卡尝试，单张网卡默认路由就只有一张网卡）
>
> 如果需要修改，修改`ds.spec.template.spec.containers.args`如下：
>
> ~~~yaml
>       containers:
>       - name: kube-flannel
>         image: quay.io/coreos/flannel:v0.11.0-amd64
>         command:
>         - /opt/bin/flanneld
>         args:
>         - --ip-masq
>         - --kube-subnet-mgr
>         - --iface=ens33
> ~~~
>
> 其中ens33为网卡名。

4. 查看pod的网络情况

~~~
kubectl get pod -n kube-system -o wide 
输出
root@Kubernetes-Master:/data/kubernetes/flannel# kubectl get po -n kube-system -o wide 
NAME                                        READY   STATUS    RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
coredns-78d4cf999f-gqq8g                    1/1     Running   0          85m     10.244.0.2       kubernetes-master   <none>           <none>
coredns-78d4cf999f-p76dh                    1/1     Running   0          85m     10.244.0.3       kubernetes-master   <none>           <none>
etcd-kubernetes-master                      1/1     Running   1          84m     192.168.17.130   kubernetes-master   <none>           <none>
kube-apiserver-kubernetes-master            1/1     Running   1          84m     192.168.17.130   kubernetes-master   <none>           <none>
kube-controller-manager-kubernetes-master   1/1     Running   1          84m     192.168.17.130   kubernetes-master   <none>           <none>
kube-flannel-ds-amd64-szr4g                 1/1     Running   0          2m24s   192.168.17.130   kubernetes-master   <none>           <none>
kube-proxy-5hsrw                            1/1     Running   1          85m     192.168.17.130   kubernetes-master   <none>           <none>
kube-scheduler-kubernetes-master            1/1     Running   1          84m     192.168.17.130   kubernetes-master   <none>           <none>
~~~

> 上面看到pod：coredns的地址为上面kubeadm init kubenrenets时定义的IP段

### 加入节点

1. 加入节点

   运行上面的`kubeadm init`命令输出的：`kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>`

~~~
kubeadm join 192.168.17.130:6443 --token 671p99.pmvkuvnxic6dcghr --discovery-token-ca-cert-hash sha256:6f6840323044e48fc21c965695a3489f75046e3f7a4b0a37da47349b33a09fce
~~~

输出：

```shell
root@Kubernetes-Node1:~# kubeadm join 192.168.17.130:6443 --token 671p99.pmvkuvnxic6dcghr --discovery-token-ca-cert-hash sha256:6f6840323044e48fc21c965695a3489f75046e3f7a4b0a37da47349b33a09fce
[preflight] Running pre-flight checks
[discovery] Trying to connect to API Server "192.168.17.130:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.17.130:6443"
[discovery] Requesting info from "https://192.168.17.130:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.17.130:6443"
[discovery] Successfully established connection with API Server "192.168.17.130:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kubernetes-node1" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

~~~shell
root@Kubernetes-Node2:~#  kubeadm join 192.168.17.130:6443 --token 671p99.pmvkuvnxic6dcghr --discovery-token-ca-cert-hash sha256:6f6840323044e48fc21c965695a3489f75046e3f7a4b0a37da47349b33a09fce
[preflight] Running pre-flight checks
[discovery] Trying to connect to API Server "192.168.17.130:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.17.130:6443"
[discovery] Requesting info from "https://192.168.17.130:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.17.130:6443"
[discovery] Successfully established connection with API Server "192.168.17.130:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kubernetes-node2" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
~~~

> 本次操作的过程中，出现了flanenl的镜像：`quay.io/coreos/flannel:v0.11.0-amd64`下载慢的问题,可以在节点上提前下载镜像（该镜像是`kube-flannel.yml`文件中image需要的镜像）。
>
> ~~~shell
> docker pull quay.io/coreos/flannel:v0.11.0-amd64
> ~~~

等待`flannel`的pod创建完毕为：`runnging`状态，验证节点状态：

~~~shell
kubectl get pod -n kube-system -l app=flannel -o wide 
##输出
NAME                          READY   STATUS    RESTARTS   AGE    IP               NODE                NOMINATED NODE   READINESS GATES
kube-flannel-ds-amd64-7q7j8   1/1     Running   0          127m   192.168.17.132   kubernetes-node2    <none>           <none>
kube-flannel-ds-amd64-f76lj   1/1     Running   0          127m   192.168.17.131   kubernetes-node1    <none>           <none>
kube-flannel-ds-amd64-pdrs8   1/1     Running   0          127m   192.168.17.130   kubernetes-master   <none>           <none>
~~~

~~~
root@Kubernetes-Master:~# kubectl get node 
NAME                STATUS   ROLES    AGE    VERSION
kubernetes-master   Ready    master   4h1m   v1.13.4
kubernetes-node1    Ready    <none>   129m   v1.13.4
kubernetes-node2    Ready    <none>   128m   v1.13.4
~~~

2. 给节点打角色标签

```shell
kubectl label node kubernetes-node1 node-role.kubernetes.io/node1=
##输出
node/kubernetes-node1 labeled
kubectl label node kubernetes-node2 node-role.kubernetes.io/node2=
##输出
node/kubernetes-node2 labeled
```

检查：

```shell
root@Kubernetes-Master:~# kubectl get node
NAME                STATUS   ROLES    AGE     VERSION
kubernetes-master   Ready    master   4h12m   v1.13.4
kubernetes-node1    Ready    node1    140m    v1.13.4
kubernetes-node2    Ready    node2    139m    v1.13.4
```

3. 让Master节点也可调度pod

~~~shell
kubectl taint nodes --all node-role.kubernetes.io/master-
##输出
node/kubernetes-master untainted
taint "node-role.kubernetes.io/master:" not found
taint "node-role.kubernetes.io/master:" not found
~~~



4. 忘记node加入集群的命令参数

- 忘记token

~~~
root@Kubernetes-Master:/data/kubernetes/caclico# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
z1wkfj.iznyn5uwmimc6csf   19h       2019-03-16T10:18:52+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
~~~

上面看到有个TTL列，表示该token的还剩多少有校时间，token的有效时间是24H，使用下面命令生成新的token：

~~~
root@Kubernetes-Master:/data/kubernetes/caclico# kubeadm token create
#输出
ze5uz7.uwgpfqwadtbm9wp6
root@Kubernetes-Master:/data/kubernetes/caclico# kubeadm token list
#输出
\TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
z1wkfj.iznyn5uwmimc6csf   19h       2019-03-16T10:18:52+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
ze5uz7.uwgpfqwadtbm9wp6   23h       2019-03-16T14:55:46+08:00   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
~~~

- 忘记hash参数：`--discovery-token-ca-cert-hash`

~~~
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# 输出
eebfe256113bee397b218ba832f412273ae734bd4686241fb910885d26efd222
~~~

### 启用IPVS



​	ipvs使用需要依赖5个内核参数和安装两个软件包。每个节点都需要。

​	记录默认的ipvs，默认的规则为：iptables

  ~~~shell
 curllocalhost:10249/proxyMode
 ##输出
 iptablesroot@Kubernetes-Master:~# 
  ~~~

> 这个命令可查看kube-proxy的转发模式

1. 内核参数添加

   内核参数需要：`ip_vs`、`ip_vs_rr`、`ip_vs_wrr`、 `ip_vs_sh`、`nf_conntrack_ipv4`

~~~shell
for i in ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4; do modprobe $i; done
~~~

2. 检查内核参数：

~~~shell
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
##输出
root@Kubernetes-Master:~# lsmod | grep -e ip_vs -e nf_conntrack_ipv4
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 147456  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack_ipv4      16384  6
nf_defrag_ipv4         16384  1 nf_conntrack_ipv4
nf_conntrack          106496  9 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4,nf_conntrack_ipv6
libcrc32c              16384  2 raid456,ip_vs
~~~

3、安装依赖软件

~~~shell
apt-get update
apt -y install ipset ipvsadm
~~~

4. 修改kube-proxy的configMap：

~~~shell
kubectl edit cm kube-proxy -n kube-system
##
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      syncPeriod: 30s
    kind: KubeProxyConfiguration
    metricsBindAddress: 127.0.0.1:10249
    mode: "ipvs"
    nodePortAddresses: null
    oomScoreAdj: -999
    portRange: ""
    resourceContainer: /kube-proxy
    udpIdleTimeout: 250ms
~~~

> 备注：修改 mode: "ipvs"

5. 重启`kube-proxy`的pod

~~~shell
kubectl get pod -n kube-system | grep kube-proxy | awk '{system("kubectl delete pod "$1" -n kube-system")}'
~~~

6. 检查`kube-proxy`的转发模式

~~~shell
curl localhost:10249/proxyMode 
##输出
ipvsroot@Kubernetes-Master:~# 
~~~

7. 查看`kube-proxy`的pod日志：

~~~shell
kubectl get po -l k8s-app=kube-proxy -n kube-system
##输出
NAME               READY   STATUS    RESTARTS   AGE
kube-proxy-56nh7   1/1     Running   0          8s
kube-proxy-rvt7w   1/1     Running   0          3s
kube-proxy-w2dqf   1/1     Running   0          5s
kubectl -n kube-system log kube-proxy-56nh7
##输出
I0412 06:17:28.639448       1 server_others.go:189] Using ipvs Proxier.
W0412 06:17:28.639926       1 proxier.go:365] IPVS scheduler not specified, use rr by default
I0412 06:17:28.640117       1 server_others.go:216] Tearing down inactive rules.
I0412 06:17:28.689497       1 server.go:464] Version: v1.13.1
I0412 06:17:28.714187       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I0412 06:17:28.714682       1 config.go:202] Starting service config controller
I0412 06:17:28.714744       1 controller_utils.go:1027] Waiting for caches to sync for service config controller
I0412 06:17:28.715123       1 config.go:102] Starting endpoints config controller
I0412 06:17:28.715274       1 controller_utils.go:1027] Waiting for caches to sync for endpoints config controller
I0412 06:17:28.815421       1 controller_utils.go:1034] Caches are synced for endpoints config controller
I0412 06:17:28.815503       1 controller_utils.go:1034] Caches are synced for service config controller
~~~

### 验证集群

1. 创建资源

~~~shell
kubectl create deployment nginx --image=nginx:alpine
##输出
deployment.apps/nginx created
~~~

2. 查看资源

~~~shell
kubectl get deploy
输出
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           10s
~~~

3. 调整资源规模

~~~shell
kubectl scale deploy/nginx --replicas=3
##输出
deployment.extensions/nginx scaled
~~~

4. 查看结果

~~~shell
root@Kubernetes-Master:~# kubectl get deployment -o wide 
NAME    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
nginx   3/3     3            3           4m53s   nginx        nginx:alpine   app=nginx
root@Kubernetes-Master:~# kubectl get po -o wide 
NAME                     READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
nginx-54458cd494-8gllz   1/1     Running   0          5m19s   10.244.1.2   kubernetes-node1    <none>           <none>
nginx-54458cd494-8w96s   1/1     Running   0          35s     10.244.2.2   kubernetes-node2    <none>           <none>
nginx-54458cd494-vp9vw   1/1     Running   0          35s     10.244.0.4   kubernetes-master   <none>           <none>
~~~

附：扩建过程controller日志信息

> ~~~shell
> root@Kubernetes-Master:~# kubectl get events -w
> LAST SEEN   TYPE     REASON              KIND         MESSAGE
> 14m         Normal   Starting            Node         Starting kube-proxy.
> 14m         Normal   Starting            Node         Starting kube-proxy.
> 14m         Normal   Starting            Node         Starting kube-proxy.
> 4m37s       Normal   Scheduled           Pod          Successfully assigned default/nginx-54458cd494-8gllz to kubernetes-node1
> 4m36s       Normal   Pulled              Pod          Container image "nginx:alpine" already present on machine
> 4m36s       Normal   Created             Pod          Created container
> 4m35s       Normal   Started             Pod          Started container
> 4m37s       Normal   SuccessfulCreate    ReplicaSet   Created pod: nginx-54458cd494-8gllz
> 4m37s       Normal   ScalingReplicaSet   Deployment   Scaled up replica set nginx-54458cd494 to 1
> 0s    Normal   ScalingReplicaSet   Deployment   Scaled up replica set nginx-54458cd494 to 3
> 0s    Normal   SuccessfulCreate   ReplicaSet   Created pod: nginx-54458cd494-8w96s
> 0s    Normal   Scheduled   Pod   Successfully assigned default/nginx-54458cd494-8w96s to kubernetes-node2
> 0s    Normal   SuccessfulCreate   ReplicaSet   Created pod: nginx-54458cd494-vp9vw
> 0s    Normal   Scheduled   Pod   Successfully assigned default/nginx-54458cd494-vp9vw to kubernetes-master
> 0s    Normal   Pulled   Pod   Container image "nginx:alpine" already present on machine
> 0s    Normal   Pulled   Pod   Container image "nginx:alpine" already present on machine
> 0s    Normal   Created   Pod   Created container
> 0s    Normal   Created   Pod   Created container
> 0s    Normal   Started   Pod   Started container
> 0s    Normal   Started   Pod   Started container
> ~~~
>
> 

### 验证kube-proxy

1. 创建SVC

```shell
kubectl expose deploy nginx --port=80 --type=NodePort
##输出
service/nginx exposed
#检查
kubectl get svc
##输出
root@Kubernetes-Master:~# kubectl get svc 
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        4h48m
nginx        NodePort    10.96.20.195   <none>        80:30057/TCP   11s

```

2. 从集群外部验证

~~~shell
root@Kubernetes-Master:~# curl http://192.168.17.131:30057
root@Kubernetes-Master:~# curl http://192.168.17.130:30057
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
~~~

3. 从集群内部验证

~~~shell
kubectl get pod -o wide
##输出
NAME                     READY   STATUS    RESTARTS   AGE    IP           NODE                NOMINATED NODE   READINESS GATES
nginx-54458cd494-8gllz   1/1     Running   0          10m    10.244.1.2   kubernetes-node1    <none>           <none>
nginx-54458cd494-8w96s   1/1     Running   0          6m5s   10.244.2.2   kubernetes-node2    <none>           <none>
nginx-54458cd494-vp9vw   1/1     Running   0          6m5s   10.244.0.4   kubernetes-master   <none>           <none>
##进入容器
kubectl exec -it nginx-54458cd494-vp9vw -- /bin/sh
##解析服务名
nslookup nginx
##输出
/ # nslookup nginx
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx
Address 1: 10.96.20.195 nginx.default.svc.cluster.local
/ # curl http://nginx
~~~

4. 查看IPVS规则

~~~shell
ipvsadm -L -n
##输出
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.17.130:30057 rr
  -> 10.244.0.4:80                Masq    1      0          0         
  -> 10.244.1.2:80                Masq    1      0          0         
  -> 10.244.2.2:80                Masq    1      0          0         
TCP  10.96.0.1:443 rr
  -> 192.168.17.130:6443          Masq    1      0          0         
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0         
  -> 10.244.0.3:53                Masq    1      0          0         
TCP  10.96.20.195:80 rr
  -> 10.244.0.4:80                Masq    1      0          0         
  -> 10.244.1.2:80                Masq    1      0          0         
  -> 10.244.2.2:80                Masq    1      0          0         
TCP  10.244.0.0:30057 rr
  -> 10.244.0.4:80                Masq    1      0          0         
  -> 10.244.1.2:80                Masq    1      0          0         
  -> 10.244.2.2:80                Masq    1      0          0         
TCP  10.244.0.1:30057 rr
  -> 10.244.0.4:80                Masq    1      0          0         
  -> 10.244.1.2:80                Masq    1      0          0         
  -> 10.244.2.2:80                Masq    1      0          0         
TCP  127.0.0.1:30057 rr
  -> 10.244.0.4:80                Masq    1      0          0         
  -> 10.244.1.2:80                Masq    1      0          0         
  -> 10.244.2.2:80                Masq    1      0          0         
TCP  172.17.0.1:30057 rr
  -> 10.244.0.4:80                Masq    1      0          0         
  -> 10.244.1.2:80                Masq    1      0          0         
  -> 10.244.2.2:80                Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          1         
  -> 10.244.0.3:53                Masq    1      0          1 
~~~



## 卸载集群

前言：

- kubectl drain 命令选项介绍

~~~shell
Options:
      --delete-local-data=false
      : Continue even if there are pods using emptyDir (local data that will be deleted when
		the node is drained).
      --dry-run=false
      : If true, only print the object that would be sent, without sending it.
      --force=false
      : Continue even if there are pods not managed by a ReplicationController, ReplicaSet, Job, DaemonSet
		or StatefulSet.
      --grace-period=-1
      : Period of time in seconds given to each pod to terminate gracefully. 
      If negative, the default value specified in the pod will be used.
      --ignore-daemonsets=false: Ignore DaemonSet-managed pods.
      --pod-selector='': Label selector to filter pods on the node
  -l, --selector='': Selector (label query) to filter on
      --timeout=0s: The length of time to wait before giving up, zero means infinite
~~~



- 在Master 节点上运行：

~~~shell
删除node2
    root@Kubernetes-Master:~# kubectl drain kubernetes-node2 --delete-local-data --force --ignore-daemonsets
##输出
node/kubernetes-node2 cordoned
WARNING: Ignoring DaemonSet-managed pods: calico-node-6gzfs, kube-proxy-rxl56
pod/coredns-78d4cf999f-k5bnt evicted
pod/nginx-54458cd494-8s252 evicted
pod/curl-66959f6557-gdk29 evicted
node/kubernetes-node2 evicted
删除node1
root@Kubernetes-Master:~# kubectl drain kubernetes-node1 --delete-local-data --force --ignore-daemonsets 
##输出
node/kubernetes-node1 cordoned
WARNING: Ignoring DaemonSet-managed pods: calico-node-bfv6r, kube-proxy-sb7kd
pod/nginx-54458cd494-hl69w evicted
pod/nginx-54458cd494-7jkp9 evicted
pod/coredns-78d4cf999f-q5kxw evicted
pod/curl-66959f6557-qj929 evicted
node/kubernetes-node1 evicted
~~~

- 查看节点：

~~~shell
root@Kubernetes-Master:~# kubectl get node 
NAME                STATUS                     ROLES    AGE    VERSION
kubernetes-master   Ready                      master   14d    v1.13.4
kubernetes-node1    Ready,SchedulingDisabled   <none>   2d5h   v1.13.4
kubernetes-node2    Ready,SchedulingDisabled   <none>   2d5h   v1.13.4
~~~

- 删除节点

~~~shell
root@Kubernetes-Master:~# kubectl delete node kubernetes-node1
node "kubernetes-node1" deleted
root@Kubernetes-Master:~# kubectl delete node kubernetes-node2
node "kubernetes-node2" deleted
~~~

- 删除网络插件

~~~shell
root@Kubernetes-Master:~# cd /data/kubernetes/caclico/
root@Kubernetes-Master:/data/kubernetes/caclico# ls
calico.yaml  rbac-kdd.yaml
root@Kubernetes-Master:/data/kubernetes/caclico# kubectl delete -f calico.yaml 
configmap "calico-config" deleted
service "calico-typha" deleted
deployment.apps "calico-typha" deleted
poddisruptionbudget.policy "calico-typha" deleted
daemonset.extensions "calico-node" deleted
serviceaccount "calico-node" deleted
customresourcedefinition.apiextensions.k8s.io "felixconfigurations.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "bgppeers.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "bgpconfigurations.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "ippools.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "hostendpoints.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "clusterinformations.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "globalnetworkpolicies.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "globalnetworksets.crd.projectcalico.org" deleted
customresourcedefinition.apiextensions.k8s.io "networkpolicies.crd.projectcalico.org" deleted
root@Kubernetes-Master:/data/kubernetes/caclico# kubectl delete -f rbac-kdd.yaml 
clusterrole.rbac.authorization.k8s.io "calico-node" deleted
clusterrolebinding.rbac.authorization.k8s.io "calico-node" deleted
~~~

### 初始化

~~~shell
root@Kubernetes-Master:/data/kubernetes/caclico# kubeadm reset 
##输出
[reset] WARNING: changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] are you sure you want to proceed? [y/N]: Y
[preflight] running pre-flight checks
[reset] Reading configuration from the cluster...
[reset] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[reset] stopping the kubelet service
[reset] unmounting mounted directories in "/var/lib/kubelet"
[reset] deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kubernetes]
[reset] deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually.
For example: 
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

~~~

### 清除iptables规则

- 三台节点都要操作

~~~shell
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
~~~

- 检查

~~~shell
root@Kubernetes-Master:/data/kubernetes/caclico# iptables -L -n -v
Chain INPUT (policy ACCEPT 151 packets, 84658 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 137 packets, 83092 bytes)
 pkts bytes target     prot opt in     out     source               destination         
~~~

