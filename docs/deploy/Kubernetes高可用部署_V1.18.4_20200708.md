# Kubernetes高可用集群部署

[TOC]



## 前言

### Kubernetes 架构简读

![](https://jimmysong.io/kubernetes-handbook/images/architecture.png)

Kubernetes主要由以下几个核心组件组成：

Master节点上：

- etcd保存了整个集群的状态；
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；

Node节点上：

- kubelet负责维护容器的生命周期，同时也负责Volume（CSI）和网络（CNI）的管理；
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；
- 网络插件：Kubernetes中将容器的联网通过插件的方式来实现



### 安装的版本

> - kubernetes： v1.18.4
> - etcd:  v3.4.7
> - flannel v0.11.0
> - docker ce: 19.03.6

### 部署的网络信息：

> - service cluster IP CIDR:100.67.0.0/16
> - cluster IP CIDR:100.68.0.0/16
> - Kubernetes IP :100.67.0.1
> - service DNS IP:100.67.0.2
> - DNS DN:

### Kubernetes  相关

> ```
> kubernetes API VIP:  192.168.137.165
> 
> kubernetes域名: k8s-api.virtual.local
> ```

### 节点信息：

|       IP        | Hostname | CPU  | Memory |       System       |         Kernel          |
| :-------------: | :------: | :--: | :----: | :----------------: | :---------------------: |
| 192.168.137.166 |  K8S-M1  |  2   |   4    | Ubuntu 16.04.4 LTS | Linux 4.4.0-116-generic |
| 192.168.137.167 |  K8S-M2  |  2   |   4    | Ubuntu 16.04.4 LTS | Linux 4.4.0-116-generic |
| 192.168.137.168 |  K8S-M3  |  2   |   4    | Ubuntu 16.04.4 LTS | Linux 4.4.0-116-generic |
| 192.168.137.169 |  K8S-M4  |  2   |   4    | Ubuntu 16.04.4 LTS | Linux 4.4.0-116-generic |
| 192.168.137.170 |  K8S-M5  |  2   |   4    | Ubuntu 16.04.4 LTS | Linux 4.4.0-116-generic |
| 192.168.137.171 |  K8S-N1  |  2   |   4    | Ubuntu 16.04.4 LTS | Linux 4.4.0-116-generic |

 ### 基础环境规整

>  以下每个机器都要规整

1. 主机名解析

~~~shell
sed -i "s/127.0.1.1/#127.0.1.1/g" /etc/hosts
echo "$(ifconfig | grep "inet addr" | cut -d":" -f2 | cut -d" " -f1 | head -1) $(hostname)" >> /etc/hosts
~~~

2. DNS解析

~~~shell
#使用公网服务器解析
cat >>/etc/resolvconf/resolv.conf.d/head <<end
nameserver 223.5.5.5
nameserver 223.6.6.6
end
resolvconf -u
~~~

3. 内核参数

> - Kubernetes v1.8+要求关闭系统Swap,若不关闭则需要修改kubelet设定参数( –fail-swap-on 设置为 false 来忽略 swap on),在`所有机器`使用以下指令关闭swap并注释掉`/etc/fstab`中swap的行,不过本次部署的机器没有挂载swap分区。
> - ipvs依赖于`nf_conntrack_ipv4`内核模块,`4.19`包括之后内核里改名为`nf_conntrack`,1.13.1之前的kube-proxy的代码里没有加判断一直用的`nf_conntrack_ipv4`,好像是1.13.1后的kube-proxy代码里增加了判断

~~~shell
#内核参数配置
cat >>/etc/sysctl.conf <<EOF
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 60
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_max_tw_buckets = 6000
net.nf_conntrack_max = 655350
net.netfilter.nf_conntrack_tcp_timeout_established = 300
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 262144
vm.swappiness = 0
vm.max_map_count=262144
net.ipv4.tcp_orphan_retries = 1
net.ipv4.ip_nonlocal_bind = 1
EOF
modprobe  ip_conntrack
sysctl -p
#添加开机启动ip_conntrack模块
vim /etc/rc.local
#
modprobe  ip_conntrack
sysctl -p
exit 0
~~~

4. 句柄数

~~~shell
#句柄参数设置规范
echo "*          hard    nproc     1030993" >>/etc/security/limits.d/20-nproc.conf 
echo "*          soft    nproc     1030993" >>/etc/security/limits.d/20-nproc.conf 
echo "*          soft    nofile    65535"  >>/etc/security/limits.d/20-nproc.conf
echo "*          hard    nofile    65535"  >>/etc/security/limits.d/20-nproc.conf
echo "*          hard    nproc     1030993" >>/etc/security/limits.conf 
echo "*          soft    nproc     1030993" >>/etc/security/limits.conf
echo "*          soft    nofile    65535"  >>/etc/security/limits.conf
echo "*          hard    nofile    65535"  >>/etc/security/limits.conf
echo "ulimit -u 1030993" >> /etc/profile
echo "ulimit -n 65535" >> /etc/profile
source /etc/profile
~~~

> 测试环境没有那么大并发，可以不用设置

5. 防火墙

~~~shell
#防火墙配置
systemctl stop ufw
systemctl disable ufw
~~~

6. 软件源

~~~shell
sed -i 's/http[^ ]*/http:\/\/mirrors.aliyun.com\/ubuntu\//g' /etc/apt/sources.list
~~~

7. 添加docker源

~~~shell
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 7EA0A9C3F273FCD8

apt-get update
~~~

8. 软件安装

~~~shell
apt-get install -y  nscd ntp ntpdate sysstat lrzsz
~~~

9. 开启缓存

~~~shell
systemctl enable nscd
~~~

10. 设置NTP 同步

> 集群是tls的，时间误差会认证失败

~~~shell
timedatectl set-timezone Asia/Shanghai
sed -i "s/pool 0.ubuntu.pool.ntp.org iburst/#pool 0.ubuntu.pool.ntp.org iburst/g" /etc/ntp.conf
sed -i "s/pool 1.ubuntu.pool.ntp.org iburst/#pool 1.ubuntu.pool.ntp.org iburst/g" /etc/ntp.conf
sed -i "s/pool 2.ubuntu.pool.ntp.org iburst/#pool 2.ubuntu.pool.ntp.org iburst/g" /etc/ntp.conf
sed -i "s/pool 3.ubuntu.pool.ntp.org iburst/#pool 3.ubuntu.pool.ntp.org iburst/g" /etc/ntp.conf
sed -i "s/pool ntp.ubuntu.com/#pool ntp.ubuntu.com/g" /etc/ntp.conf

echo "server ntp1.aliyun.com"  >> /etc/ntp.conf
echo "server ntp2.aliyun.com"  >> /etc/ntp.conf
systemctl enable ntp
systemctl restart ntp
#检查
ntpq -p
~~~

### 安装ansible

> K8S-M1安装就行了，这里安装的ansible只是为了检验某些服务状态的时候使用，因为很长时间没有部署K8S了，怕使用ansible出错不好排查。所以以下基本都是手码苦工搞一下

~~~shell
#主机：K8S-M1
apt -y install sshpass ansible
~~~

> 密码请写真实密码

~~~shell
[master]
192.168.137.167 ansible_ssh_user="root" ansible_ssh_pass="x"
192.168.137.168 ansible_ssh_user="root" ansible_ssh_pass="x"
192.168.137.169 ansible_ssh_user="root" ansible_ssh_pass="x"
192.168.137.170 ansible_ssh_user="root" ansible_ssh_pass="x"
[node]
192.168.137.167 ansible_ssh_user="root" ansible_ssh_pass="x"
192.168.137.168 ansible_ssh_user="root" ansible_ssh_pass="x"
192.168.137.169 ansible_ssh_user="root" ansible_ssh_pass="x"
192.168.137.170 ansible_ssh_user="root" ansible_ssh_pass="x"
192.168.137.171 ansible_ssh_user="root" ansible_ssh_pass="x"
~~~

测试联通性

~~~shell
ansible master -m ping 
~~~

### 参考文档：

1. [架构解读：https://jimmysong.io/kubernetes-handbook/concepts/](https://jimmysong.io/kubernetes-handbook/concepts/)
2. [Kubernetes中的网络：https://jimmysong.io/kubernetes-handbook/concepts/networking.html](https://jimmysong.io/kubernetes-handbook/concepts/networking.html)
3. 

## 1. 安装环境准备

### 1. 域名配置

> 配置k8s集群域名对应的VIP地址，后续的很多证书都是使用的这个域名，所以需要每个节点都加这个hosts解析

执行设备：master+node

~~~shell
cat >> /etc/hosts <<EOF
#kubernetes域名解析
192.168.137.165  k8s-api.virtual.local   k8s-api
192.168.137.166 k8s-m1
192.168.137.167 k8s-m2
192.168.137.168 k8s-m3
192.168.137.169 k8s-m4
192.168.137.170 k8s-m5
192.168.137.171 k8s-n1
EOF
~~~

### 2.目录准备

执行设备：master + node

~~~shell
mkdir -p /opt/kubernetes/{cfg,bin,ssl,log}
~~~

### 环境变量准备

执行设备：master + node

~~~shell
echo 'PATH=$PATH:/opt/kubernetes/bin' >/etc/profile.d/k8s.sh
source /etc/profile.d/k8s.sh
#检查：
echo $PATH
~~~

### 配置集群全局变量

> 我在网上参考人家的资料，可以一个组件整理成一个脚本，可以在每个组件部署的时候，

执行设备： master+node

~~~shell
cat > /opt/kubernetes/bin/env.sh <<EOF
# TLS Bootstrapping 使用的Token，可以使用命令 head -c 16 /dev/urandom | od -An -t x | tr -d ' ' 生成
BOOTSTRAP_TOKEN="e6b5333ed136d5d23a9809ee543b4246"
#当前部署节点的IP
export NODE_IP=`ifconfig | grep "inet addr" | cut -d":" -f2 | cut -d" " -f1 | head -1`
# 建议使用未用的网段来定义服务网段和Pod 网段
# 服务网段(Service CIDR)，部署前路由不可达，部署后集群内部使用IP:Port可达
SERVICE_CIDR="100.67.0.0/16"
# Pod 网段(Cluster CIDR)，部署前路由不可达，部署后路由可达(flanneld 保证)
CLUSTER_CIDR="100.68.0.0/16"
# 服务端口范围(NodePort Range)
NODE_PORT_RANGE="30000-32766"
# etcd集群服务地址列表 (注意换行符)
ETCD_ENDPOINTS="https://192.168.137.166:2379,https://192.168.137.167:2379,https://192.168.137.168:2379,https://192.168.137.169:2379,https://192.168.137.170:2379"
# flanneld 网络配置前缀
FLANNEL_ETCD_PREFIX="/kubernetes/network"
# kubernetes 服务IP(预先分配，一般为SERVICE_CIDR中的第一个IP)
CLUSTER_KUBERNETES_SVC_IP="100.67.0.1"
# 集群 DNS 服务IP(从SERVICE_CIDR 中预先分配)
CLUSTER_DNS_SVC_IP="100.67.0.2"
# 集群 DNS 域名
CLUSTER_DNS_DOMAIN="cluster.local."
# MASTER API Server 地址
MASTER_URL="k8s-api.virtual.local"
export KUBE_APISERVER="https://k8s-api.virtual.local"
EOF
source /opt/kubernetes/bin/env.sh
#检查：
echo $NODE_IP
echo $ETCD_ENDPOINTS
echo $KUBE_APISERVER
~~~

## 2. 二进制包准备：

### 1. 准备目录

执行设备：K8S-M1

~~~shell
mkdir -p /home/xxsun5/pkg/kubernetes_1.18.4
cd /home/xxsun5/pkg/kubernetes_1.18.4
~~~

### 2. 下载二进制包

执行设备：K8S-M1

~~~shell
cd /home/xxsun5/pkg/kubernetes_1.18.4
wget https://dl.k8s.io/v1.18.4/kubernetes.tar.gz
wget https://dl.k8s.io/v1.18.4/kubernetes-client-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.18.4/kubernetes-node-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.18.4/kubernetes-server-linux-amd64.tar.gz
~~~

> 不知道是不是网络环境的原因，这几台VMware的虚拟机下载不下来，我用生产的一台服务器下载的资源，复制到了目录：`cd /home/xxsun5/pkg/kubernetes_1.18.4`
>
> 补充： 现在这个地址被重定向到一个谷歌得存储地址了，被墙了，直接下载不下来 
>
> ​                                                       																								--2020年7月6日

执行设备：K8S-M1

~~~shell
#检查
cd /home/xxsun5/pkg/kubernetes_1.18.4
ll -h
total 454M
drwxr-xr-x 2 root root 4.0K Jun 18 11:18 ./
drwxr-xr-x 3 root root 4.0K Jun 18 11:09 ../
-rw-r--r-- 1 root root  13M Jun 18 11:18 kubernetes-client-linux-amd64.tar.gz
-rw-r--r-- 1 root root  94M Jun 18 11:18 kubernetes-node-linux-amd64.tar.gz
-rw-r--r-- 1 root root 347M Jun 18 11:18 kubernetes-server-linux-amd64.tar.gz
-rw-r--r-- 1 root root 455K Jun 18 11:18 kubernetes.tar.gz
~~~

> 如果有检验下载文件正确性的需求，可使用，命令：`sha512sum` 的校验码和官方对比。
>
> 官方校验码入口：[官方校验码传送门](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md#downloads-for-v1184)

执行设备：K8S-M1

~~~shell
# sha512sum *
69344ab18d8f9608374cd4994ffbd878a9738b14793a41a25dfbedb0bf9a6174605507f92e7032930700c79ccc12ca899570a099d9db624d4c65580aefedbe0c  kubernetes-client-linux-amd64.tar.gz
b01c34c0303116a2c7a579fec5bcd19d76fa605c6ec9fa7c9885e669437911365cf63c8be381aebab666f833ff9099be024fb139d3ddc50e5f9b6702352b5a3c  kubernetes-node-linux-amd64.tar.gz
e85fbe9aa255cabcf58b4c18fa666d6a85effa0fc9c78d0d150abf3f89bfd13fcd55163caf02188b095310e01f49e1f32464c9e87f43cebe1043868436e0e751  kubernetes-server-linux-amd64.tar.gz
8d2cec9d026bbed016f004c23e205e234bcd40072cda81e805ecebe6e8cc8e4b5f1685c9dc57640edc3c5c67e09ac362bfa9bae1b654fcf425d3e4e184b5b46f  kubernetes.tar.gz
~~~

### 3.解压缩二进制压缩包

执行设备：K8S-M1

~~~shell
cd /home/xxsun5/pkg/kubernetes_1.18.4
tar zvxf kubernetes.tar.gz 
tar zvxf kubernetes-server-linux-amd64.tar.gz 
tar zvxf kubernetes-client-linux-amd64.tar.gz
tar zvxf kubernetes-node-linux-amd64.tar.gz
~~~

## 3. CA证书准备

### 3.1安装cfssl

执行设备：K8S-M1

~~~shell
mkdir /home/xxsun5/pkg/cfssl
cd /home/xxsun5/pkg/cfssl
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl*
cp cfssl-certinfo_linux-amd64 /opt/kubernetes/bin/cfssl-certinfo
cp cfssljson_linux-amd64  /opt/kubernetes/bin/cfssljson
cp cfssl_linux-amd64  /opt/kubernetes/bin/cfssl
#复制cfssl命令文件到其他节点
ansible master -m copy -a 'src=/opt/kubernetes/bin/cfssl dest=/opt/kubernetes/bin/ mode=0755'
ansible master -m copy -a 'src=/opt/kubernetes/bin/cfssljson dest=/opt/kubernetes/bin/ mode=0755'
ansible master -m copy -a 'src=/opt/kubernetes/bin/cfssl-certinfo dest=/opt/kubernetes/bin/ mode=0755'
~~~

### 3.2 初始化cfssl

执行设备：K8S-M1

~~~shell
mkdir /home/xxsun5/pkg/ssl
cd /home/xxsun5/pkg/ssl
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
~~~

### 3.3 创建用来生成CA文件的JSON配置文件

执行设备：K8S-M1

~~~shell
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
~~~

### 3.4 创建用来生成 CA 证书签名请求（CSR）的 JSON 配置文件

执行设备：K8S-M1

~~~shell
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
~~~

### 3.5 生成CA证书（ca.pem）和密钥（ca-key.pem）

执行设备：K8S-M1

~~~shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
#输出
2020/06/18 14:57:44 [INFO] generating a new CA key and certificate from CSR
2020/06/18 14:57:44 [INFO] generate received request
2020/06/18 14:57:44 [INFO] received CSR
2020/06/18 14:57:44 [INFO] generating key: rsa-2048
2020/06/18 14:57:46 [INFO] encoded CSR
2020/06/18 14:57:46 [INFO] signed certificate with serial number 52114562040824832857571870825592956084353721220
root@K8S-M1:/home/xxsun5/pkg/ssl# ll ca*
-rw-r--r-- 1 root root  292 Jun 18 14:57 ca-config.json
-rw-r--r-- 1 root root 1001 Jun 18 14:57 ca.csr
-rw-r--r-- 1 root root  208 Jun 18 14:57 ca-csr.json
-rw------- 1 root root 1675 Jun 18 14:57 ca-key.pem
-rw-r--r-- 1 root root 1359 Jun 18 14:57 ca.pem
~~~

### 3.6 分发证书

> CA证书需要分发到集群中的所有节点.

执行设备：K8S-M1

~~~shell
cp ca.csr ca.pem ca-key.pem ca-config.json /opt/kubernetes/ssl
ansible node -m copy -a 'src=/opt/kubernetes/ssl/ca.csr dest=/opt/kubernetes/ssl/'
ansible node -m copy -a 'src=/opt/kubernetes/ssl/ca.pem dest=/opt/kubernetes/ssl/'
ansible node -m copy -a 'src=/opt/kubernetes/ssl/ca-key.pem dest=/opt/kubernetes/ssl/'
ansible node -m copy -a 'src=/opt/kubernetes/ssl/ca-config.json dest=/opt/kubernetes/ssl/'
~~~

执行设备：K8S-M1

~~~shell
#检查
ansible node -m shell -a 'ls -lh /opt/kubernetes/ssl/'
~~~

## 4.部署etcd集群

### 4.1 下载etcd二进制包

> 官方GitHub仓库可以下载最新版本的二进制文件：[官方GitHub仓库传送门](https://github.com/coreos/etcd/releases)

执行设备：K8S-M1

~~~shell
mkdir /home/xxsun5/pkg/etcd-v3.4.7
cd /home/xxsun5/pkg/etcd-v3.4.7
wget https://github.com/coreos/etcd/releases/download/v3.4.7/etcd-v3.4.7-linux-amd64.tar.gz
tar zxf etcd-v3.4.7-linux-amd64.tar.gz
cd etcd-v3.4.7-linux-amd64
cp etcd etcdctl /opt/kubernetes/bin/
ansible master -m copy -a 'src=/opt/kubernetes/bin/etcd dest=/opt/kubernetes/bin/ mode=0755'
ansible master -m copy -a 'src=/opt/kubernetes/bin/etcdctl dest=/opt/kubernetes/bin/ mode=0755'
~~~

### 4.2 设置etcd环境变量

> 每个master的etcd的NODE_NAME和NODE_IP不一致，所以需要相应修改etcd环境变量的相关信息。

执行设备：K8S-M1

~~~shell
cat > /opt/kubernetes/bin/etcd-env.sh <<EOF
# 当前部署的机器名称(随便定义，只要能区分不同机器即可,例如etcd节点2改为etcd02)
export NODE_NAME=etcd166

# 当前部署的机器IP（其他节点执行，要相应修改为本机IP）
export NODE_IP=192.168.137.166

# etcd 集群所有机器 IP
export NODE_IPS="192.168.137.166 192.168.137.167 192.168.137.168 192.168.137.169 192.168.137.170" 

# etcd 集群间通信的IP和端口(注意换行符)
export ETCD_NODES=etcd166=https://192.168.137.166:2380,etcd167=https://192.168.137.167:2380,etcd168=https://192.168.137.168:2380,etcd169=https://192.168.137.169:2380,etcd170=https://192.168.137.170:2380

# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
#source /usr/k8s/bin/env.sh
EOF
#使环境变量生效：
source /opt/kubernetes/bin/etcd-env.sh
~~~

执行设备：K8S-M2

~~~shell
cat > /opt/kubernetes/bin/etcd-env.sh <<EOF
# 当前部署的机器名称(随便定义，只要能区分不同机器即可,例如etcd节点2改为etcd02)
export NODE_NAME=etcd167

# 当前部署的机器IP（其他节点执行，要相应修改为本机IP）
export NODE_IP=192.168.137.167

# etcd 集群所有机器 IP
export NODE_IPS="192.168.137.166 192.168.137.167 192.168.137.168 192.168.137.169 192.168.137.170" 

# etcd 集群间通信的IP和端口(注意换行符)
export ETCD_NODES=etcd166=https://192.168.137.166:2380,etcd167=https://192.168.137.167:2380,etcd168=https://192.168.137.168:2380,etcd169=https://192.168.137.169:2380,etcd170=https://192.168.137.170:2380

# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
#source /usr/k8s/bin/env.sh
EOF
#使环境变量生效：
source /opt/kubernetes/bin/etcd-env.sh
~~~

执行设备：K8S-M3

~~~shell
cat > /opt/kubernetes/bin/etcd-env.sh <<EOF
# 当前部署的机器名称(随便定义，只要能区分不同机器即可,例如etcd节点2改为etcd02)
export NODE_NAME=etcd168

# 当前部署的机器IP（其他节点执行，要相应修改为本机IP）
export NODE_IP=192.168.137.168

# etcd 集群所有机器 IP
export NODE_IPS="192.168.137.166 192.168.137.167 192.168.137.168 192.168.137.169 192.168.137.170" 

# etcd 集群间通信的IP和端口(注意换行符)
export ETCD_NODES=etcd166=https://192.168.137.166:2380,etcd167=https://192.168.137.167:2380,etcd168=https://192.168.137.168:2380,etcd169=https://192.168.137.169:2380,etcd170=https://192.168.137.170:2380

# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
#source /usr/k8s/bin/env.sh
EOF
#使环境变量生效：
source /opt/kubernetes/bin/etcd-env.sh
~~~

执行设备：K8S-M4

~~~shell
cat > /opt/kubernetes/bin/etcd-env.sh <<EOF
# 当前部署的机器名称(随便定义，只要能区分不同机器即可,例如etcd节点2改为etcd02)
export NODE_NAME=etcd169

# 当前部署的机器IP（其他节点执行，要相应修改为本机IP）
export NODE_IP=192.168.137.169

# etcd 集群所有机器 IP
export NODE_IPS="192.168.137.166 192.168.137.167 192.168.137.168 192.168.137.169 192.168.137.170" 

# etcd 集群间通信的IP和端口(注意换行符)
export ETCD_NODES=etcd166=https://192.168.137.166:2380,etcd167=https://192.168.137.167:2380,etcd168=https://192.168.137.168:2380,etcd169=https://192.168.137.169:2380,etcd170=https://192.168.137.170:2380

# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
#source /usr/k8s/bin/env.sh
EOF
#使环境变量生效：
source /opt/kubernetes/bin/etcd-env.sh
~~~

执行设备：K8S-M5

~~~shell
cat > /opt/kubernetes/bin/etcd-env.sh <<EOF
# 当前部署的机器名称(随便定义，只要能区分不同机器即可,例如etcd节点2改为etcd02)
export NODE_NAME=etcd170

# 当前部署的机器IP（其他节点执行，要相应修改为本机IP）
export NODE_IP=192.168.137.170

# etcd 集群所有机器 IP
export NODE_IPS="192.168.137.166 192.168.137.167 192.168.137.168 192.168.137.169 192.168.137.170" 

# etcd 集群间通信的IP和端口(注意换行符)
export ETCD_NODES=etcd166=https://192.168.137.166:2380,etcd167=https://192.168.137.167:2380,etcd168=https://192.168.137.168:2380,etcd169=https://192.168.137.169:2380,etcd170=https://192.168.137.170:2380

# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
#source /usr/k8s/bin/env.sh
EOF
#使环境变量生效：
source /opt/kubernetes/bin/etcd-env.sh
~~~

### 4.3 创建etcd证书签名请求

> 文件：` etcd-csr.json` 里的变量：`$NODE_IP`，每个节点都不一样，所以需要在master上都执行，而这个文件需要在：生成etcd证书的时候用到，所以，每个节点的证书也都不一样

执行设备：master

```json
mkdir -p /home/xxsun5/pkg/ssl/
cd /home/xxsun5/pkg/ssl/
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
"$NODE_IP"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

### 4.4 生产etcd证书和私钥

> 文件：` etcd-csr.json` 不一样，则产生的证书也不一样，所以需要每个master都执行

执行设备：master

~~~shell
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
  -ca-key=/opt/kubernetes/ssl/ca-key.pem \
  -config=/opt/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
#输出
2020/06/18 16:13:24 [INFO] generate received request
2020/06/18 16:13:24 [INFO] received CSR
2020/06/18 16:13:24 [INFO] generating key: rsa-2048
2020/06/18 16:13:25 [INFO] encoded CSR
2020/06/18 16:13:25 [INFO] signed certificate with serial number 623735729874192758104159302945863913985361481592
2020/06/18 16:13:25 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
# 生成以下证书文件
root@K8S-M1:/home/xxsun5/pkg/ssl# ll etcd*
-rw-r--r-- 1 root root 1045 Jun 18 16:13 etcd.csr
-rw-r--r-- 1 root root  255 Jun 18 16:12 etcd-csr.json
-rw------- 1 root root 1675 Jun 18 16:13 etcd-key.pem
-rw-r--r-- 1 root root 1419 Jun 18 16:13 etcd.pem

~~~

### 4.5 分发证书

> 将证书移动到/opt/kubernetes/ssl目录下

执行设备：master

~~~shell
cp etcd*.pem /opt/kubernetes/ssl
~~~

### 4.6 检查etcd的证书有效期

执行设备：master任意一台

~~~shell
# openssl x509 -text -in etcd.pem | grep Not
            Not Before: Jun 18 08:12:00 2020 GMT
            Not After : Jun 16 08:12:00 2030 GMT
~~~

### 4.7创建etcd配置文件

> 每个节点的环境变量不同，需要分节点执行，最好在执行前检查一下每个节点的环境变量

执行设备：master

~~~shell
cat >/opt/kubernetes/cfg/etcd.yml <<EOF
name: ${NODE_NAME}
data-dir: /var/lib/etcd/default.etcd
listen-peer-urls: https://${NODE_IP}:2380
listen-client-urls: https://${NODE_IP}:2379,https://127.0.0.1:2379

advertise-client-urls: https://${NODE_IP}:2379
initial-advertise-peer-urls: https://${NODE_IP}:2380
initial-cluster: ${ETCD_NODES}
initial-cluster-token: k8s-etcd-cluster
initial-cluster-state: new

client-transport-security:
  cert-file: /opt/kubernetes/ssl/etcd.pem
  key-file: /opt/kubernetes/ssl/etcd-key.pem
  client-cert-auth: false
  trusted-ca-file: /opt/kubernetes/ssl/ca.pem
  auto-tls: false

peer-transport-security:
  cert-file: /opt/kubernetes/ssl/etcd.pem
  key-file: /opt/kubernetes/ssl/etcd-key.pem
  client-cert-auth: false
  trusted-ca-file: /opt/kubernetes/ssl/ca.pem
  auto-tls: false

debug: false
logger: zap
log-outputs: [stderr]
EOF
~~~

### 4.8 创建etcd存储目录

执行设备：K8S-M1

~~~shell
mkdir /var/lib/etcd
ansible master -m shell -a "mkdir /var/lib/etcd"
~~~

### 4.9创建etcd 的systemd管理文件

执行设备：K8S-M1

```shell
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
Documentation=https://github.com/etcd-io/etcd
Conflicts=etcd.service
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
LimitNOFILE=65536
Restart=on-failure
RestartSec=5s
TimeoutStartSec=0
ExecStart=/opt/kubernetes/bin/etcd --config-file=/opt/kubernetes/cfg/etcd.yml

[Install]
WantedBy=multi-user.target
EOF
```

> 分发至其他节点

执行设备：K8S-M1

~~~shell
ansible master -m copy -a 'src=/etc/systemd/system/etcd.service dest=/etc/systemd/system/'
~~~

### 4.10 启动etcd

执行设备：master

~~~shell
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
#输出
● etcd.service - Etcd Server
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-06-18 22:02:09 CST; 46min ago
     Docs: https://github.com/etcd-io/etcd
 Main PID: 14292 (etcd)
   CGroup: /system.slice/etcd.service
           └─14292 /opt/kubernetes/bin/etcd

Jun 18 22:02:09 K8S-M1 etcd[14292]: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
Jun 18 22:02:09 K8S-M1 systemd[1]: Started Etcd Server.
Jun 18 22:39:28 K8S-M1 systemd[1]: Started Etcd Server.
Jun 18 22:46:18 K8S-M1 systemd[1]: etcd.service: Dependency Conflicts=etcd.service dropped
Jun 18 22:46:24 K8S-M1 systemd[1]: etcd.service: Dependency Conflicts=etcd.service dropped
Jun 18 22:46:24 K8S-M1 systemd[1]: etcd.service: Dependency Conflicts=etcd.service dropped
Jun 18 22:46:24 K8S-M1 systemd[1]: Started Etcd Server.
Jun 18 22:48:47 K8S-M1 systemd[1]: etcd.service: Dependency Conflicts=etcd.service dropped
Jun 18 22:48:47 K8S-M1 systemd[1]: etcd.service: Dependency Conflicts=etcd.service dropped
Jun 18 22:48:47 K8S-M1 systemd[1]: Started Etcd Server.

~~~

### 4.11 检查

执行设备：master任意一台

~~~shell
#查看进程
# ss -ntlp|grep etcd
LISTEN     0      65535  192.168.137.166:2379                     *:*                   users:(("etcd",pid=14764,fd=7))
LISTEN     0      65535  127.0.0.1:2379                     *:*                   users:(("etcd",pid=14764,fd=6))
LISTEN     0      65535  192.168.137.166:2380                     *:*                   users:(("etcd",pid=14764,fd=5))
~~~

~~~shell
#检查集群状态
ETCDCTL_API=3 etcdctl --write-out=table --cacert=/opt/kubernetes/ssl/ca.pem --cert=/opt/kubernetes/ssl/etcd.pem --key=/opt/kubernetes/ssl/etcd-key.pem --endpoints=https://192.168.137.166:2379,https://192.168.137.167:2379,https://192.168.137.168:2379,https://192.168.137.169:2379,https://192.168.137.170:2379 endpoint health
~~~

~~~shell
#输出
+------------------------------+--------+-------------+-------+
|           ENDPOINT           | HEALTH |    TOOK     | ERROR |
+------------------------------+--------+-------------+-------+
| https://192.168.137.169:2379 |   true | 56.504011ms |       |
| https://192.168.137.168:2379 |   true | 57.683532ms |       |
| https://192.168.137.166:2379 |   true | 58.657652ms |       |
| https://192.168.137.167:2379 |   true |  61.96827ms |       |
| https://192.168.137.170:2379 |   true | 71.275535ms |       |
+------------------------------+--------+-------------+-------+
~~~

~~~shell
#这个命令应该和上面一样的
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 /opt/kubernetes/bin/etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/opt/kubernetes/ssl/ca.pem \
  --cert=/opt/kubernetes/ssl/etcd.pem \
  --key=/opt/kubernetes/ssl/etcd-key.pem \
  endpoint health; done
~~~

### 启动失败踩坑记录

- 问题表现： systemctl start etcd失败 

- 排错：

  1. 查看启动日志：journalctl -f -u etcd.service

     ~~~shell
     错误1：
     Jun 18 16:54:21 K8S-M2 etcd[9315]: health check for peer a384bd43720259b6 could not connect: x509: certificate signed by unknown authority
     错误2
     Jun 18 16:54:22 K8S-M2 etcd[9315]: rejected connection from "192.168.137.168:9608" (error "remote error: tls: bad certificate", ServerName "")
     错误3:
     Jun 18 17:02:27 K8S-M2 etcd[9392]: publish error: etcdserver: request timed out
     ~~~

  2. 查看服务状态：systemctl status etcd.service

  ![ef28fe851bfd4f829d71851cd9486f7.png](https://i.loli.net/2020/06/19/EPajHgzb5LuqtrS.png)

- 问题解决

  按照如下配置：

  ~~~shell
  # cat /opt/kubernetes/cfg/etcd.conf 
  #[member]
  ETCD_NAME="etcd170"
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  #ETCD_SNAPSHOT_COUNTER="10000"
  #ETCD_HEARTBEAT_INTERVAL="100"
  #ETCD_ELECTION_TIMEOUT="1000"
  ETCD_LISTEN_PEER_URLS="https://192.168.137.170:2380"
  ETCD_LISTEN_CLIENT_URLS="https://192.168.137.170:2379,http://127.0.0.1:2379"
  #ETCD_MAX_SNAPSHOTS="5"
  #ETCD_MAX_WALS="5"
  #ETCD_CORS=""
  #[cluster]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.137.170:2380"
  # if you use different ETCD_NAME (e.g. test),
  # set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
  ETCD_INITIAL_CLUSTER="etcd166=https://192.168.137.166:2380,etcd167=https://192.168.137.167:2380,etcd168=https://192.168.137.168:2380,etcd169=https://192.168.137.169:2380,etcd170=https://192.168.137.170:2380"
  ETCD_INITIAL_CLUSTER_STATE="new"
  ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
  ETCD_ADVERTISE_CLIENT_URLS="https://192.168.137.170:2379"
  #[security]
  CLIENT_CERT_AUTH="true"
  ETCD_CA_FILE="/opt/kubernetes/ssl/ca.pem"
  ETCD_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
  ETCD_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
  PEER_CLIENT_CERT_AUTH="true"
  ETCD_PEER_CA_FILE="/opt/kubernetes/ssl/ca.pem"
  ETCD_PEER_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
  ETCD_PEER_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
  ~~~

  当时部署`K8S：1.10`的时候，使用`etcd version : 3.2.24` 版本，使用加载环境变量的方式加载配置选项，应该是版本更迭之后，不支持这种部署方式了。这个加载环境变量是在service文件里指定的：

  ~~~shell
  EnvironmentFile=-/opt/kubernetes/cfg/etcd.conf
  ~~~
  

这样部署没有启动成功，具体**成功方式的按照上面的步骤来：** 设置配置文件、启动--conf加载配置文件

### 参考文档

1. [ETCD官方文档](https://etcd.io/docs/v3.4.0/learning/)
2. [ETCD github示例配置文档](https://github.com/etcd-io/etcd/blob/master/etcd.conf.yml.sample)
3. [Kubernetes v1.18.2 二进制高可用部署](https://www.jianshu.com/p/ddd7a73d7b2a)

## 5. 高可用组件部署

### 5.1部署haproxy

> 按照下面的方式在master机器上安装kube-apiserver、kube-controller-manager、kube-scheduler，但是需要手动指定访问的6443和8080端口的，因为我们的域名k8s-api.virtual.local对应的master节点直接通过http 和https 还不能访问，这里我们使用haproxy 来代替请求。就是我们需要将http默认的80端口请求转发到apiserver的8080端口，将https默认的443端口请求转发到apiserver的6443端口，所以我们这里使用haproxy来做请求转发。

#### 5.1.1 安装haproxy

执行设备： master

```shell
apt-get -y install haproxy
```

#### 5.1.2 配置haproxy：haproxy.cfg

> 通过配置http 和https 两种代理方式，实现对apiserver的访问
>
> 如下配置，通过https的访问将请求转发给apiserver 的6443端口

执行设备：K8S-M1

~~~shell
vim /etc/haproxy/haproxy.cfg
#在文件底部添加如下内

#config for kubernetes
listen stats
  bind    *:9000
  mode    http
  stats   enable
  stats   hide-version
  stats   uri       /stats
  stats   refresh   30s
  stats   realm     Haproxy\ Statistics
  stats   auth      Admin:xxsun5

frontend k8s-https-api
    bind 192.168.137.165:443
    mode tcp
    option tcplog
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }
    default_backend k8s-https-api

backend k8s-https-api
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server k8s-api-1 192.168.137.166:6443 check
    server k8s-api-2 192.168.137.167:6443 check
    server k8s-api-3 192.168.137.168:6443 check
    server k8s-api-4 192.168.137.169:6443 check
    server k8s-api-5 192.168.137.170:6443 check
~~~

~~~shell
#分发
ansible master -m copy -a 'src=/etc/haproxy/haproxy.cfg dest=/etc/haproxy/haproxy.cfg'
~~~



#### 5.1.2 启动haproxy

执行设备：master

~~~shell
systemctl enable haproxy
systemctl restart haproxy
systemctl status haproxy
~~~

~~~shell
ss -ntlp|grep 9000
~~~



> 我们可以通过上面9000端口监控我们的haproxy的运行状态（ http://192.168.137.165:9000/stats ），目前没有部署keepalived的，192.168.137.165 vip暂时不可用，可以临时替换VIP地址为本机IP（ 如192.168.137.166 ）测试验证，部署高合用节点时bind的IP必须配置问VIP地址。



### 5.2 部署keepalived

> KeepAlived 是一个高可用方案，通过 VIP（即虚拟 IP）和心跳检测来实现高可用。其原理是存在一组（至少两台）服务器，分别赋予 Master、Backup 两个角色，默认情况下Master 会绑定VIP 到自己的网卡上，对外提供服务。Master、Backup 会在一定的时间间隔向对方发送心跳数据包来检测对方的状态，这个时间间隔一般为 2 秒钟，如果Backup 发现Master 宕机，那么Backup 会发送ARP 包到网关，把VIP 绑定到自己的网卡，此时Backup 对外提供服务，实现自动化的故障转移，当Master 恢复的时候会重新接管服务。非常类似于路由器中的虚拟路由器冗余协议（VRRP）

#### 5.2.1 验证内核参数

> 开启内核路由转发

执行设备： master

~~~shell
vi /etc/sysctl.conf
#添加以下内容
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1

#使生效
sysctl -p
#验证是否生效
cat /proc/sys/net/ipv4/ip_forward
~~~

> **备注：以上已在基础环境规范里调整了，验证一下就可以了**

#### 5.2.2 安装keepalived

执行设备： master

```shell
apt-get -y install keepalived
```

#### 5.2.3  修改配置文件：keepalived.conf

> 

执行设备： K8S-M1

~~~shell
vim /etc/keepalived/keepalived.conf
i
! Configuration File for keepalived

global_defs {
   notification_email {
   }
   router_id kube_api
}

vrrp_script check_k8s {
    # 自身状态检测
    script "/etc/keepalived/chk_k8s_master.sh"
    interval 2
    #监测失败惩罚分数，要让失败之后的惩罚力度要小于其他节点的：priority
    weight -25
    fall 3  
    rise 2
}

vrrp_instance haproxy-vip {
    # 使用组播通信，默认是组播通信（ 本机的IP ）
    mcast_src_ip 192.168.137.166

    # 初始化状态（由于是用 priority 来选举 master，所以这句没什么用处）
    state MASTER
    # 虚拟ip 绑定的网卡 （这里根据你自己的实际情况选择网卡，用于发送 VRRP 包）
    interface ens33
    # 此ID 要与Backup 配置一致
    virtual_router_id 240
    # 默认启动优先级，要比Backup 大点，但要控制量，保证自身状态检测生效,其余节点减2
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxsun5
    }
    virtual_ipaddress {
        # 虚拟ip 地址
        192.168.137.165
    }
    track_script {
        check_k8s
    }
}
~~~

执行设备：K8S-M2

~~~shell
vim /etc/keepalived/keepalived.conf
i
! Configuration File for keepalived

global_defs {
   notification_email {
   }
   router_id kube_api
}

vrrp_script check_k8s {
    # 自身状态检测
    script "/etc/keepalived/chk_k8s_master.sh"
    interval 2
    #监测失败惩罚分数，要让失败之后的惩罚力度要小于其他节点的：priority
    weight -25
    fall 3  
    rise 2
}

vrrp_instance haproxy-vip {
    # 使用组播通信，默认是组播通信（ 本机的IP ）
    mcast_src_ip 192.168.137.167

    # 初始化状态（由于是用 priority 来选举 master，所以这句没什么用处）
    state BACKUP
    # 虚拟ip 绑定的网卡 （这里根据你自己的实际情况选择网卡，用于发送 VRRP 包）
    interface ens33
    # 此ID 要与Backup 配置一致
    virtual_router_id 240
    # 默认启动优先级，要比Backup 大点，但要控制量，保证自身状态检测生效,其余节点减2
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxsun5
    }
    virtual_ipaddress {
        # 虚拟ip 地址
        192.168.137.165
    }
    track_script {
        check_k8s
    }
}
~~~

> 修改3处：`mcast_src_ip`、`state`、`priority 99`

执行设备：K8S-M3

~~~shell
vim /etc/keepalived/keepalived.conf
i
! Configuration File for keepalived

global_defs {
   notification_email {
   }
   router_id kube_api
}

vrrp_script check_k8s {
    # 自身状态检测
    script "/etc/keepalived/chk_k8s_master.sh"
    interval 2
    #监测失败惩罚分数，要让失败之后的惩罚力度要小于其他节点的：priority
    weight -25
    fall 3  
    rise 2
}

vrrp_instance haproxy-vip {
    # 使用组播通信，默认是组播通信（ 本机的IP ）
    mcast_src_ip 192.168.137.168

    # 初始化状态（由于是用 priority 来选举 master，所以这句没什么用处）
    state BACKUP
    # 虚拟ip 绑定的网卡 （这里根据你自己的实际情况选择网卡，用于发送 VRRP 包）
    interface ens33
    # 此ID 要与Backup 配置一致
    virtual_router_id 240
    # 默认启动优先级，要比Backup 大点，但要控制量，保证自身状态检测生效,其余节点减2
    priority 97
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxsun5
    }
    virtual_ipaddress {
        # 虚拟ip 地址
        192.168.137.165
    }
    track_script {
        check_k8s
    }
}
~~~

>  修改2处：`mcast_src_ip`、`priority `

执行设备：K8S-M4

~~~shell
vim /etc/keepalived/keepalived.conf
i
! Configuration File for keepalived

global_defs {
   notification_email {
   }
   router_id kube_api
}

vrrp_script check_k8s {
    # 自身状态检测
    script "/etc/keepalived/chk_k8s_master.sh"
    interval 2
    #监测失败惩罚分数，要让失败之后的惩罚力度要小于其他节点的：priority
    weight -25
    fall 3  
    rise 2
}

vrrp_instance haproxy-vip {
    # 使用组播通信，默认是组播通信（ 本机的IP ）
    mcast_src_ip 192.168.137.169

    # 初始化状态（由于是用 priority 来选举 master，所以这句没什么用处）
    state BACKUP
    # 虚拟ip 绑定的网卡 （这里根据你自己的实际情况选择网卡，用于发送 VRRP 包）
    interface ens33
    # 此ID 要与Backup 配置一致
    virtual_router_id 240
    # 默认启动优先级，要比Backup 大点，但要控制量，保证自身状态检测生效,其余节点减2
    priority 95
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxsun5
    }
    virtual_ipaddress {
        # 虚拟ip 地址
        192.168.137.165
    }
    track_script {
        check_k8s
    }
}
~~~

>  修改2处：`mcast_src_ip`、`priority`

执行设备：K8S-M5

~~~shell
vim /etc/keepalived/keepalived.conf
i
! Configuration File for keepalived

global_defs {
   notification_email {
   }
   router_id kube_api
}

vrrp_script check_k8s {
    # 自身状态检测
    script "/etc/keepalived/chk_k8s_master.sh"
    interval 2
    #监测失败惩罚分数，要让失败之后的惩罚力度要小于其他节点的：priority
    weight -25
    fall 3  
    rise 2
}

vrrp_instance haproxy-vip {
    # 使用组播通信，默认是组播通信（ 本机的IP ）
    mcast_src_ip 192.168.137.170

    # 初始化状态（由于是用 priority 来选举 master，所以这句没什么用处）
    state BACKUP
    # 虚拟ip 绑定的网卡 （这里根据你自己的实际情况选择网卡，用于发送 VRRP 包）
    interface ens33
    # 此ID 要与Backup 配置一致
    virtual_router_id 240
    # 默认启动优先级，要比Backup 大点，但要控制量，保证自身状态检测生效,其余节点减2
    priority 93
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxsun5
    }
    virtual_ipaddress {
        # 虚拟ip 地址
        192.168.137.165
    }
    track_script {
        check_k8s
    }
}
~~~

>  修改2处：`mcast_src_ip`、`priority`

>以上配置是使用组播通信，只有两个节点可以配置单播通信，如下配置：
>
>```
>vrrp_instance haproxy-vip {
># 使用单播通信，默认是组播通信
>   unicast_src_ip 192.168.76.129
>   unicast_peer {
>   192.168.76.130
>   }
>```

#### 5.2.4 设置检测脚本

>用于监测haproxy的存活

执行设备： master

~~~shell
vi /etc/keepalived/chk_k8s_master.sh
i
#!/bin/bash

status=$(ps aux|grep haproxy | grep -v grep | grep -v bash | wc -l)
if [ "${status}" = "0" ]; then
    echo "haproxy restart"
    #/etc/init.d/haproxy start
    systemctl start haproxy

    sleep 2

    status2=$(ps aux|grep haproxy | grep -v grep | grep -v bash |wc -l)

    if [ "${status2}" = "0"  ]; then
      #/etc/init.d/keepalived stop
      exit 1
    else
      exit 0 
    fi
else
      exit 0
fi
~~~

#### 5.2.5 赋予脚本执行权限

执行设备： K8S-M1

~~~shell
chmod 755 /etc/keepalived/chk_k8s_master.sh
~~~

~~~shell
ansible master -m copy -a 'src=/etc/keepalived/chk_k8s_master.sh dest=/etc/keepalived/chk_k8s_master.sh mode=0755'
~~~



#### 5.2.6 启动keepalive

执行设备：master

~~~shell
systemctl enable keepalived
systemctl restart keepalived
systemctl status keepalived
~~~

执行设备：master

~~~shell
#查看keepalive的日志
journalctl -f -u keepalived
~~~



#### 5.2.7 验证 vip

1. ping

   执行设备：任意

   ~~~shell
   root@k8s-N1:~# ping 192.168.137.165 -c 4
   PING 192.168.137.165 (192.168.137.165) 56(84) bytes of data.
   64 bytes from 192.168.137.165: icmp_seq=1 ttl=64 time=0.397 ms
   64 bytes from 192.168.137.165: icmp_seq=2 ttl=64 time=0.480 ms
   64 bytes from 192.168.137.165: icmp_seq=3 ttl=64 time=0.454 ms
   64 bytes from 192.168.137.165: icmp_seq=4 ttl=64 time=0.469 ms
   
   --- 192.168.137.165 ping statistics ---
   4 packets transmitted, 4 received, 0% packet loss, time 3000ms
   rtt min/avg/max/mdev = 0.397/0.450/0.480/0.031 ms
   ~~~

2. ip a

   > 因为K8S-M1是keepalive的master，所以虚IP在该设备上

   执行设备： K8S-M1


   ~~~shell
   root@K8S-M1:/home/xxsun5/pkg/ssl# ip addr 
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host 
          valid_lft forever preferred_lft forever
   2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
       link/ether 00:0c:29:03:86:99 brd ff:ff:ff:ff:ff:ff
       inet 192.168.137.166/24 brd 192.168.137.255 scope global ens33
          valid_lft forever preferred_lft forever
       inet 192.168.137.165/32 scope global ens33
          valid_lft forever preferred_lft forever
       inet6 fe80::20c:29ff:fe03:8699/64 scope link 
          valid_lft forever preferred_lft forever
   ~~~

3. 访问haproxy的web界面

   执行客户端： chrome浏览器

   ![image.png](https://i.loli.net/2020/06/19/FjOHhQn9vgAafid.png)

#### 问题记录

启动完keepalive之后，查看所有master的haproxy的9000端口，发现都不在了，重启后`ss -ntlp|grep 9000` 查看又好了，访问页面也正常了

## 6.Master节点部署

> 如最初前言里描述，K8S的master节点，需要部署的组件包括：apiserver、controller manager、scheduler。

### 6.1准备二进制程序



~~~shell
cd /home/xxsun5/pkg/kubernetes_1.18.4/kubernetes/server/bin
cp kube-apiserver kube-controller-manager kube-scheduler kubectl /opt/kubernetes/bin/
ansible master -m copy -a 'src=/opt/kubernetes/bin/kube-apiserver dest=/opt/kubernetes/bin/ mode=0755'
ansible master -m copy -a 'src=/opt/kubernetes/bin/kube-controller-manager dest=/opt/kubernetes/bin/ mode=0755'
ansible master -m copy -a 'src=/opt/kubernetes/bin/kube-scheduler dest=/opt/kubernetes/bin/ mode=0755'
ansible master -m copy -a 'src=/opt/kubernetes/bin/kubectl dest=/opt/kubernetes/bin/ mode=0755'
~~~

### 6.2部署kube-apiserver

#### 6.2.1验证环境变量

执行设备：master

~~~shell
echo $NODE_IP
echo ${MASTER_URL}
echo ${CLUSTER_KUBERNETES_SVC_IP}
echo ${ETCD_ENDPOINTS}
~~~

只是简答的验证几个，要是不通过，请执行以下部署

~~~shell
# 当前部署的master机器IP
export NODE_IP=172.21.52.31
source /opt/kubernetes/bin/env.sh
~~~

#### 6.2.2 创建生成CSR的JSON配置文件

> 因为NODE_IP不一样，所以需要所有master节点都执行

执行设备：master

~~~shell
cd /home/xxsun5/pkg/ssl/
cat >kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "$aliyundm.mail.mcubelab.comP",
    "${MASTER_URL}",
    "${CLUSTER_KUBERNETES_SVC_IP}",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
~~~

#### 6.2.3生成 kubernetes 证书和私钥

执行设备：master

~~~shell
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

cp kubernetes*.pem /opt/kubernetes/ssl/
~~~

#### 6.2.4 验证证书的有效期

执行设备：master任意

~~~shell
 openssl x509 -text -in /opt/kubernetes/ssl/kubernetes.pem  |grep Not
            Not Before: Jun 19 08:27:00 2020 GMT
            Not After : Jun 17 08:27:00 2030 GMT
~~~

#### 6.2.5  创建kube-apiserver 使用的客户端 token 文件

执行设备： K8S-M1

~~~shell
cat > /opt/kubernetes/ssl/bootstrap-token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
ansible master -m copy -a 'src=/opt/kubernetes/ssl/bootstrap-token.csv dest=/opt/kubernetes/ssl/'
~~~

#### 6.2.6 创建 metrics-server 证书

> 这个证书会在`kubel top node/pod`的时候使用，而且kube-apiserver需要指定该证书的位置：
>
> `--proxy-client-cert-file=/opt/kubernetes/ssl/metrics-server.pem`
>
> `--proxy-client-key-file=/opt/kubernetes/ssl/metrics-server-key.pem `

执行设备： K8S-M1

1. 创建csr的json文件

   ~~~shell
   cd /home/xxsun5/pkg/ssl
   
   # 注意: "CN": "system:metrics-server" 一定是这个，因为后面授权时用到这个名称，否则会报禁止匿名访问
   cat > metrics-server-csr.json <<EOF
   {
     "CN": "system:metrics-server",
     "hosts": [],
     "key": {
       "algo": "rsa",
       "size": 2048
     },
     "names": [
       {
         "C": "CN",
         "ST": "BeiJing",
         "L": "BeiJing",
         "O": "k8s",
         "OU": "system"
       }
     ]
   }
   EOF
   ~~~

   2. 生成证书

      ~~~shell
      cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem  \
      -ca-key=/opt/kubernetes/ssl/ca-key.pem \
      -config=/opt/kubernetes/ssl/ca-config.json \
      -profile=kubernetes metrics-server-csr.json | cfssljson -bare metrics-server
      #输出
      2020/07/02 14:12:22 [INFO] generate received request
      2020/07/02 14:12:22 [INFO] received CSR
      2020/07/02 14:12:22 [INFO] generating key: rsa-2048
      2020/07/02 14:12:23 [INFO] encoded CSR
      2020/07/02 14:12:23 [INFO] signed certificate with serial number 473245127458074681819904282212899057552829014774
      2020/07/02 14:12:23 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
      websites. For more information see the Baseline Requirements for the Issuance and Management
      of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
      specifically, section 10.2.3 ("Information Requirements").
      ~~~

   3. 分发证书

      ~~~shell
      cp metrics-server-key.pem metrics-server.pem /opt/kubernetes/ssl/
      ansible master -m copy -a 'src=/opt/kubernetes/ssl/metrics-server.pem dest=/opt/kubernetes/ssl/'
      ansible master -m copy -a 'src=/opt/kubernetes/ssl/metrics-server-key.pem dest=/opt/kubernetes/ssl/'
      ~~~

      

#### 6.2.7 创建kube-apiserver 的systemd文件

> NODE_IP不一样，需要分开执行

执行设备：master

~~~shell
cat  > /etc/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kubernetes/bin/kube-apiserver \\
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \\
  --bind-address=$NODE_IP \\
  --authorization-mode=Node,RBAC \\
  --runtime-config=rbac.authorization.k8s.io/v1 \\
  --kubelet-https=true \\
  --anonymous-auth=false \\
  --enable-bootstrap-token-auth \\
  --token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv \\
  --service-cluster-ip-range=$SERVICE_CIDR \\
  --service-node-port-range=$NODE_PORT_RANGE \\
  --kubelet-client-certificate=/opt/kubernetes/ssl/kubernetes.pem \\
  --kubelet-client-key=/opt/kubernetes/ssl/kubernetes-key.pem \\
  --tls-cert-file=/opt/kubernetes/ssl/kubernetes.pem \\
  --tls-private-key-file=/opt/kubernetes/ssl/kubernetes-key.pem \\
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \\
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
  --etcd-cafile=/opt/kubernetes/ssl/ca.pem \\
  --etcd-certfile=/opt/kubernetes/ssl/kubernetes.pem \\
  --etcd-keyfile=/opt/kubernetes/ssl/kubernetes-key.pem \\
  --etcd-servers=$ETCD_ENDPOINTS \\
  --requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
  --requestheader-extra-headers-prefix=X-Remote-Extra- \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --proxy-client-cert-file=/opt/kubernetes/ssl/metrics-server.pem \\
  --proxy-client-key-file=/opt/kubernetes/ssl/metrics-server-key.pem \\
  --allow-privileged=true \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/opt/kubernetes/log/api-audit.log \\
  --audit-policy-file=/opt/kubernetes/cfg/audit-policy.yaml \\
  --event-ttl=1h \\
  --v=2 \\
  --logtostderr=false \\
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
~~~

#### 6.2.8 配置审查日志策略

执行设备：K8S-M1

~~~yaml
cat  > /opt/kubernetes/cfg/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"
EOF
~~~

~~~shell
#分发至其他节点
ansible master -m copy -a 'src=/opt/kubernetes/cfg/audit-policy.yaml dest=/opt/kubernetes/cfg/'
~~~

#### 6.2.9 启动API Server服务

执行设备： master

~~~shell
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
~~~

~~~shell
#查看日志
journalctl -f -u kube-apiserver.service
~~~

~~~shell
#检查进程
# ss -ntlp|grep apiserver
LISTEN     0      65535  192.168.137.166:6443                     *:*                   users:(("kube-apiserver",pid=66017,fd=7))
LISTEN     0      65535  127.0.0.1:8080                     *:*                   users:(("kube-apiserver",pid=66017,fd=6))
~~~

#### 参考文档：

1. [kube-apiserver参数官方文档](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)
2. [Kubernetes v1.18.2 二进制高可用部署](https://www.jianshu.com/p/ddd7a73d7b2a)

### 6.3部署 Controller Manager 服务

> kubelet 发起的 CSR 请求都是由 kube-controller-manager 来做实际签署的,所有使用的证书都是根证书的密钥对 。

#### 6.3.1 准备Controller Manager的systemd文件

执行设备： master

~~~shell
cat > /etc/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-controller-manager \\
  --bind-address=127.0.0.1 \\
  --master=http://127.0.0.1:8080 \\
  --allocate-node-cidrs=true \\
  --service-cluster-ip-range=$SERVICE_CIDR \\
  --cluster-cidr=$CLUSTER_CIDR \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \\
  --experimental-cluster-signing-duration=86700h \\
  --feature-gates=RotateKubeletClientCertificate=true \\
  --leader-elect=true \\
  --v=2 \\
  --logtostderr=false \\
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
~~~

~~~shell
ansible master -m copy -a 'src=/etc/systemd/system/kube-controller-manager.service dest=/etc/systemd/system/'
~~~

> --address 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器
>
> --master=http://${MASTER_URL}:8080：使用http(非安全端口)与 kube-apiserver 通信，需要下面的haproxy安装成功后才能去掉8080端口。
>
> --cluster-cidr 指定 Cluster 中 Pod 的 CIDR 范围，该网段在各 Node 间必须路由可达(flanneld保证)
>
> --service-cluster-ip-range 参数指定 Cluster 中 Service 的CIDR范围，该网络在各 Node 间必须路由不可达，必须和 kube-apiserver 中的参数一致
>
> --cluster-signing-* 指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥
>
> --root-ca-file 用来对 kube-apiserver 证书进行校验，指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件
>
> --leader-elect=true 部署多台机器组成的 master 集群时选举产生一处于工作状态的 kube-controller-manager 进程
>
> --experimental-cluster-signing-duration=87600h0m0s 批准证书的时间是10年
>
> --feature-gates=RotateKubeletClientCertificate=true 支持自动更新证书

#### 6.3.2 启动Controller Manager

执行设备：master

~~~shell
systemctl daemon-reload
systemctl enable kube-controller-manager.service
systemctl start kube-controller-manager
systemctl status kube-controller-manager
~~~

#### 参考文档：

1. [Kubelet 证书自动续期](https://juejin.im/post/5dd35cd6f265da0be92d220e)
2. [kube-controller-manager参数官方文档](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

### 6.4部署Kubernetes Scheduler

#### 6.4.1准备Schedule的systemd的文件

执行设备：K8S-M1

~~~shell
cat > /etc/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-scheduler \\
  --bind-address=127.0.0.1 \\
  --master=http://127.0.0.1:8080 \\
  --leader-elect=true \\
  --v=2 \\
  --logtostderr=false \\
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
~~~

~~~shell
ansible master -m copy -a 'src=/etc/systemd/system/kube-scheduler.service dest=/etc/systemd/system/'
~~~

> --bind-address 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器
>
> --master=http://${MASTER_URL}:8080：使用http(非安全端口)与 kube-apiserver 通信，需要下面的haproxy启动成功后才能去掉8080端口
>
> --leader-elect=true 部署多台机器组成的 master 集群时选举产生一处于工作状态的 kube-controller-manager 进程



#### 6.4.2 启动Scheduler

执行设备： master

~~~shell
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
~~~

## 7.部署kubectl客户端

### 7.1 检查环境变量

执行设备： K8S-M1

~~~shell
echo ${KUBE_APISERVER}
#输出
https://k8s-api.virtual.local
~~~

### 7.2 创建 admin 证书签名请求

执行设备： K8S-M1

~~~shell
cd /home/xxsun5/pkg/ssl
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
~~~

### 7.3 生成 admin 证书和私钥：

执行设备： K8S-M1

~~~shell
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes admin-csr.json | cfssljson -bare admin
# 输出
2020/06/19 21:05:10 [INFO] generate received request
2020/06/19 21:05:10 [INFO] received CSR
2020/06/19 21:05:10 [INFO] generating key: rsa-2048
2020/06/19 21:05:11 [INFO] encoded CSR
2020/06/19 21:05:11 [INFO] signed certificate with serial number 91676413805353865340695452978129397832670108498
2020/06/19 21:05:11 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
~~~

### 7.4 分发证书

执行设备： K8S-M1

~~~shell
cp admin*.pem /opt/kubernetes/ssl/
#分发至其他节点
ansible master -m copy -a 'src=/opt/kubernetes/ssl/admin-key.pem dest=/opt/kubernetes/ssl/'
ansible master -m copy -a 'src=/opt/kubernetes/ssl/admin.pem dest=/opt/kubernetes/ssl/'
~~~

### 7.5 设置集群参数

执行设备： K8S-M1

~~~shell
kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kubernetes/ssl/ca.pem \
--embed-certs=true --server=$KUBE_APISERVER
#输出
Cluster "kubernetes" set.
~~~

### 7.6 设置客户端认证参数

执行设备： K8S-M1

~~~shell
kubectl config set-credentials admin \
   --client-certificate=/opt/kubernetes/ssl/admin.pem \
   --embed-certs=true \
   --client-key=/opt/kubernetes/ssl/admin-key.pem
#shuchu 
User "admin" set.
~~~

### 7.7 设置上下文参数

执行设备： K8S-M1

```
kubectl config set-context kubernetes \
   --cluster=kubernetes \
   --user=admin
#输出
Context "kubernetes" created.
```

### 7.8 设置默认上下文

执行设备： K8S-M1

~~~shell
kubectl config use-context kubernetes
#输出
Switched to context "kubernetes".
~~~

> 生成的kubeconfig 被保存到 ~/.kube/config 文件

### 7.9 验证集群

执行设备： K8S-M1

~~~shell
# kubectl get cs
#输出
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-3               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-4               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
~~~

~~~shell
# kubectl version
#输出客户端和server端的版本
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.4", GitCommit:"c96aede7b5205121079932896c4ad89bb93260af", GitTreeState:"clean", BuildDate:"2020-06-17T11:41:22Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.4", GitCommit:"c96aede7b5205121079932896c4ad89bb93260af", GitTreeState:"clean", BuildDate:"2020-06-17T11:33:59Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
~~~

##  8. Node节点部署

> 安装node节点的时候，节点上的进程是按照flannel -> docker -> kubelet -> kube-proxy的顺序启动的，但是本次安装的时候，没有按照这个方式，所以多了很多需要重启之前前面组件的步骤

### 8.1 Dokcer 部署

> ​		docker 版本的选择应该按照官方实验过的、声明支持的版本，这个比较稳妥；不过小版本应该没啥问题。请跟踪 Kubernetes 发行说明中经过验证的 Docker 最新版本变化
>
> ​		在官方查看K8s支持的docker版本 https://github.com/kubernetes/kubernetes 里进对应版本的changelog里搜`The list of validated docker versions remain`（我去搜了下，最新的docker版本声明在1.15的changelog里有表述，1.18里没有）
>
> ​		不过在官方的中文文档里也有推荐表述：https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#systemd

#### 8.1.1 下载指定版本docker

执行设备： node

~~~shell
#查看可安装的版本
apt-cache madison docker-ce
~~~

~~~shell
#安装
apt-get install -y docker-ce=5:19.03.6~3-0~ubuntu-xenial
~~~

#### 8.1.2 配置docker镜像加速

执行设备：K8S-M1

~~~shell
cat <<EOF >/etc/docker/daemon.json
{
        "registry-mirrors": [
                "https://registry.docker-cn.com"
        ],
   "max-concurrent-downloads": 10,
        "insecure-registries": [
                "172.21.52.62:5000"
        ]
          "graph": "/data/docker"
}
EOF
~~~

> 后面kubelet的启动pod需要：google_containers/pause-amd64:3.0 这个镜像，这个暂时使用线上老仓库的

~~~shell
#分发配置文件
ansible node -m copy -a 'src=/etc/docker/daemon.json dest=/etc/docker/daemon.json'
~~~



#### 8.1.3 启动docker

执行设备： node

~~~shell
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
systemctl status docker
~~~

#### 8.1.4 验证docker

执行设备： node

~~~shell
# docker version
Client: Docker Engine - Community
 Version:           19.03.11
 API version:       1.40
 Go version:        go1.13.10
 Git commit:        42e35e61f3
 Built:             Mon Jun  1 09:12:41 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.6
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.16
  Git commit:       369ce74a3c
  Built:            Thu Feb 13 01:26:38 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.2.13
  GitCommit:        7ad184331fa3e55e52b890ea95e65ba581ae3429
 runc:
  Version:          1.0.0-rc10
  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
~~~

> 部署docker的时候需注意：
>
> 1. 需要加好软件源
> 2. 后续部署好flannel的时候，需要调整systemd的启动参数，需要再重启docker

### 8.2 kubelet部署

#### 8.2.1 二进制包准备

执行设备：K8S-M1

~~~shell
cd /home/xxsun5/pkg/kubernetes_1.18.4/kubernetes/node/bin
cp kubelet kube-proxy /opt/kubernetes/bin
ansible node -m copy -a 'src=/opt/kubernetes/bin/kubelet dest=/opt/kubernetes/bin/ mode=0755'
ansible node -m copy -a 'src=/opt/kubernetes/bin/kube-proxy dest=/opt/kubernetes/bin/ mode=0755'
~~~

#### 8.2.2 创建角色绑定

执行设备：K8S-M1

~~~shell
 cd /home/xxsun5/pkg/ssl/
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
#输出
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
~~~

#### 8.2.3 创建 kubelet bootstrapping kubeconfig 文件 设置集群参数

执行设备：K8S-M1

~~~shell
#验证环境变量
echo $KUBE_APISERVER
#输出
https://k8s-api.virtual.local
#设置集群参数
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=$KUBE_APISERVER \
   --kubeconfig=bootstrap.kubeconfig
#输出
Cluster "kubernetes" set.
~~~

#### 8.2.4 设置客户端认证参数

执行设备：K8S-M1

~~~shell
#验证环境变量
echo $BOOTSTRAP_TOKEN
#输出
e6b5333ed136d5d23a9809ee543b4246
#设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
   --token=$BOOTSTRAP_TOKEN \
   --kubeconfig=bootstrap.kubeconfig
#输出
User "kubelet-bootstrap" set.
~~~

#### 8.2.5 设置上下文参数

执行设备：K8S-M1

~~~shell
kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
#输出
Context "default" created.
~~~

#### 8.2.6 设置默认上下文

执行设备：K8S-M1

~~~shell
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
#输出
Switched to context "default".
~~~

#### 8.2.7 分发配置文件

执行设备：K8S-M1

~~~shell
cp bootstrap.kubeconfig /opt/kubernetes/cfg
ansible node -m copy -a 'src=/opt/kubernetes/cfg/bootstrap.kubeconfig dest=/opt/kubernetes/cfg/'
ansible node -m copy -a 'src=/opt/kubernetes/ssl/ca.pem dest=/opt/kubernetes/ssl/'
~~~

#### 8.2.8 创建kubelet目录

执行设备：K8S-M1

~~~shell
ansible node -m shell -a 'mkdir /var/lib/kubelet'
~~~

#### 8.2.9创建Kubelet的配置文件

执行设备：node

~~~shell
echo ${HOSTNAME}
cat <<EOF >/opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=true \\
--log-dir=/opt/kubernetes/log \\
--v=2 \\
--hostname-override=${HOSTNAME} \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=172.21.52.62:5000/gcr.io/google_containers/pause-amd64:3.0"
EOF
~~~

#### 8.2.10 Kubelet的yml文件

执行设备：node

~~~shell
echo ${NODE_IP}
echo ${CLUSTER_DNS_SVC_IP}
echo ${CLUSTER_DNS_DOMAIN}
cat <<EOF >/opt/kubernetes/cfg/kubelet-config.yml
kind: KubeletConfiguration # 使用对象
apiVersion: kubelet.config.k8s.io/v1beta1 # api版本
address: ${NODE_IP} # 监听地址
port: 10250 # 当前kubelet的端口
readOnlyPort: 10255 # kubelet暴露的端口
cgroupDriver: cgroupfs # 驱动，要于docker info显示的驱动一致
clusterDNS:
  - ${CLUSTER_DNS_SVC_IP}
clusterDomain: ${CLUSTER_DNS_DOMAIN}  # 集群域
failSwapOn: false # 关闭swap

# 身份验证
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem

# 授权
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s

# Node 资源保留
evictionHard:
  imagefs.available: 15%
  memory.available: 1G
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s

# 镜像删除策略
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s

# 旋转证书
rotateCertificates: true # 旋转kubelet client 证书
featureGates:
  RotateKubeletServerCertificate: true
  RotateKubeletClientCertificate: true

maxOpenFiles: 1000000
maxPods: 110
EOF
~~~

#### 8.1.11 创建kubelet的systemd文件

执行设备：node

~~~shell
cat <<EOF >/etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
RestartSec=5s
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
~~~

#### 8.2.12 启动kubelet

执行设备：node

~~~shell
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
~~~

#### 8.2.13 验证服务

执行设备：node

~~~shell
#查看日志
journalctl -fxeu kubelet
#查看进程
ps -ef|grep kubelet
#查看端口
ss -ntlp|grep kubelet
~~~

> - 如果报：`kubelet.go:2267 node "xxx" not found`,是后面需要集群批准kubelet的csr请求
>
>   

#### 8.2.14 查看CSR请求

执行设备：K8S-M1

~~~shell
kubectl get csr
#示例输出
# kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR           CONDITION
csr-5k9qv   5m53s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
csr-8kzjv   69s     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
csr-jnr4c   16m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
csr-lx9nk   14m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
~~~

#### 8.2.15 自动批准CSR请求

执行设备：K8S-M1

~~~shell
#创建自动批准相关 CSR 请求的 ClusterRole
cat <<EOF >/opt/kubernetes/deploy/tls-instructs-csr.yaml 
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]
EOF
cd /opt/kubernetes/deploy/
kubectl apply -f tls-instructs-csr.yaml
~~~

~~~shell
#自动批准 kubelet-bootstrap 用户 TLS bootstrapping 首次申请证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-approve-csr --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient --user=kubelet-bootstrap
#自动批准 system:nodes 组用户更新 kubelet 自身与 apiserver 通讯证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient --group=system:nodes
#自动批准 system:nodes 组用户更新 kubelet 10250 api 端口证书的 CSR 请求
kubectl create clusterrolebinding node-server-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeserver --group=systemnodes
~~~

~~~shell
#这个为手动批准csr请求
kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
~~~

~~~shell
# kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR           CONDITION
csr-5k9qv   9m15s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
csr-8kzjv   4m31s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
csr-jnr4c   20m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
csr-lx9nk   18m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

~~~

#### 8.2.16 查看节点

执行设备：K8S-M1

~~~shell
# kubectl get node -o wide 
NAME     STATUS     ROLES    AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-n1   NotReady   <none>   28m   v1.18.4   192.168.137.171   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
~~~

> 手动批准请求，节点状态为NotReady  是因为还没有安装网络插件，kubelet和api还无法通信；
>
> 自动批准证书，节点状态为ready，这时kubelet和api可以通信。

#### 8.2.17 验证Kubelet的证书时间

执行设备：K8S-M1

~~~shell
# openssl x509 -text -in kubelet-client-current.pem |grep Not
            Not Before: Jun 22 07:27:30 2020 GMT
            Not After : Jun 17 10:34:00 2025 GMT
~~~

> kube-controller-manager 的启动选项里有效时间是：10年，不知道这里为什么只有5年

#### 参考文档：

1. [Kubernetes v1.18.2 二进制高可用部署](https://www.jianshu.com/p/ddd7a73d7b2a)
2. [Kubelet 证书自动续期](https://juejin.im/post/5dd35cd6f265da0be92d220e)
3. [kubelet参数官方文档](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)

### 8.3 部署kube-proxy

>

#### 8.3.1 下载安装ipvs

执行设备：node

~~~shell
apt-get -y install ipvsadm ipset conntrack
~~~

#### 8.3.2 创建 kube-proxy 证书请求

执行设备：K8S-M1

~~~shell
cd /home/xxsun5/pkg/ssl/
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
~~~

#### 8.3.3 生成证书

执行设备：K8S-M1

~~~shell
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
#输出
2020/06/22 16:12:41 [INFO] generate received request
2020/06/22 16:12:41 [INFO] received CSR
2020/06/22 16:12:41 [INFO] generating key: rsa-2048
2020/06/22 16:12:42 [INFO] encoded CSR
2020/06/22 16:12:42 [INFO] signed certificate with serial number 265019604312520738499663691594821687543917746280
2020/06/22 16:12:42 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

~~~

#### 8.3.4 分发证书

执行设备：K8S-M1

~~~shell
cp kube-proxy*.pem /opt/kubernetes/ssl/
ansible node  -m copy -a 'src=/opt/kubernetes/ssl/kube-proxy.pem dest=/opt/kubernetes/ssl/'
ansible node  -m copy -a 'src=/opt/kubernetes/ssl/kube-proxy-key.pem dest=/opt/kubernetes/ssl/'
~~~

#### 8.3.5 创建kube-proxy配置文件

执行设备：K8S-M1

~~~shell
echo $KUBE_APISERVER
#设置集群参数
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=$KUBE_APISERVER \
   --kubeconfig=kube-proxy.kubeconfig
#输出
Cluster "kubernetes" set.
#设置客户端认证参数
kubectl config set-credentials kube-proxy \
   --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
   --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig
#输出
User "kube-proxy" set.
## 设置上下文参数
kubectl config set-context default \
   --cluster=kubernetes \
   --user=kube-proxy \
   --kubeconfig=kube-proxy.kubeconfig
#输出
Context "default" created.
# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
#输出
Switched to context "default".
~~~

#### 8.3.6 分发kubeconfig配置文件到所有node节点

执行设备：K8S-M1

~~~shell
cp kube-proxy.kubeconfig /opt/kubernetes/cfg/ 
ansible node -m copy -a 'src=/opt/kubernetes/cfg/kube-proxy.kubeconfig dest=/opt/kubernetes/cfg/'
~~~

#### 8.3.7 创建工作目录

执行设备：K8S-M1

~~~shell
mkdir /var/lib/kube-proxy
ansible node -m shell -a 'mkdir /var/lib/kube-proxy'
~~~

####  8.3.8 创建kube-proxy的conf文件

执行设备：node

~~~shell
cat <<EOF >/opt/kubernetes/cfg/kube-proxy.conf
KUBE_PROXY_OPTS="--logtostderr=true \\
--v=2 \\
--log-dir=/opt/kubernetes/log \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF
~~~

#### 8.3.9 创建kube-proxy的yml文件

执行设备： node

~~~shell
echo ${NODE_IP}
echo ${HOSTNAME}
echo ${SERVICE_CIDR}
cat <<EOF >/opt/kubernetes/cfg/kube-proxy-config.yml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
address: ${NODE_IP} # 监听地址
metricsBindAddress: 0.0.0.0:10249 # 监控指标地址,监控获取相关信息 就从这里获取
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig # 读取配置文件
hostnameOverride: ${HOSTNAME} # 注册到k8s的节点名称唯一
clusterCIDR: ${SERVICE_CIDR} # service IP范围

# 使用 ipvs 模式
mode: ipvs # ipvs 模式
ipvs:
  scheduler: "rr"
  syncPeriod:: "5s"
  minSyncPeriod:  "5s"
iptables:  
  masqueradeAll: true
EOF
~~~

#### 8.3.10 创建kube-proxy的systemd文件

执行设备： node

~~~shell
cat <<EOF >/etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
EnvironmentFile=-/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF
~~~

#### 8.3.11 启动kube-proxy

执行设备： node

~~~shell
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
~~~

#### 8.3.12 验证服务

执行设备： node

~~~shell
#验证转发模式
curl localhost:10249/proxyMode
#查看ipvs转发规则
ipvsadm -L -n
#查看日志
journalctl -fexu kube-proxy
~~~

~~~shell
# 报了一下info信息，问题不大
Jun 22 18:20:19 K8S-M1 kube-proxy[104527]: I0622 18:20:19.519281  104527 proxier.go:1848] Not using `--random-fully` in the MASQUERADE rule for iptables because the local version of iptables does not support it
~~~

#### 参考资料

1. [Kubernetes v1.18.2 二进制高可用部署](https://www.jianshu.com/p/ddd7a73d7b2a)

### 8.4 部署flannel网络组件

> 之前部署flannel是通过：etcdctl v2去写pod子网，而且flannel不支持etcd v3
>
> 本次部署是通过：不走etcd v2 api下二进制跑flannel

#### 8.4.1 准备flannel二进制软件包

执行设备：K8S-M1

> 官方地址： [传送门](https://github.com/coreos/flannel/releases)

~~~shell
cd /home/xxsun5/pkg
wget https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
~~~

~~~shell
mkdir flannel
tar -zxvf flannel-v0.11.0-linux-amd64.tar.gz -C flannel
~~~

#### 8.4.2 分发二进制软件包

执行设备：K8S-M1

~~~shell
cd /home/xxsun5/pkg/flannel
cp flanneld mk-docker-opts.sh /opt/kubernetes/bin/
ansible node -m copy -a 'src=/opt/kubernetes/bin/flanneld dest=/opt/kubernetes/bin/ mode=0755'
ansible node -m copy -a 'src=/opt/kubernetes/bin/mk-docker-opts.sh dest=/opt/kubernetes/bin/ mode=0755'
~~~



#### 8.4.3 创建工作目录

执行设备：K8S-M1

```
mkdir -p /etc/cni/net.d
mkdir /run/flannel
mkdir /etc/kube-flannel/
ansible node -m shell -a 'mkdir  -p /etc/cni/net.d'
ansible node -m shell -a 'mkdir /run/flannel'
ansible node -m shell -a 'mkdir /etc/kube-flannel/'
```

#### 8.4.4 创建flannel集群角色

执行设备：K8S-M1

~~~shell
cat <<EOF >/opt/kubernetes/cfg/kube-flannel-role.yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups: ['extensions']
    resources: ['podsecuritypolicies']
    verbs: ['use']
    resourceNames: ['psp.flannel.unprivileged']
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
EOF
~~~

> 确认没有flannel集群角色后应用yaml

~~~shell
#应用yaml
cd /opt/kubernetes/cfg
kubectl get clusterrole | grep flannel || kubectl apply -f kube-flannel-role.yaml
#输出
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
~~~



#### 8.4.5 设置flanneld.kubeconfig文件集群参数

执行设备：K8S-M1

~~~shell
cd /home/xxsun/ssl
echo ${KUBE_APISERVER}
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=flanneld.kubeconfig
#输出
Cluster "kubernetes" set.
~~~

#### 8.4.6 配置上下文参数

> 

执行设备：K8S-M1

~~~shell
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=kubernetes \
  --kubeconfig=flanneld.kubeconfig
#输出
Context "kubernetes" created.
~~~

#### 8.4.7 配置认证参数

> 下面的secret和jwt_token 是应用kube-flannel-role.yaml之后才有的

执行设备：K8S-M1

~~~shell
SECRET=$(kubectl -n kube-system get sa/flannel   --output=jsonpath='{.secrets[0].name}')
JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET  --output=jsonpath='{.data.token}' | base64 -d)
kubectl config set-credentials kubernetes \
  --token=${JWT_TOKEN} \
  --kubeconfig=flanneld.kubeconfig 
#输出  
User "kubernetes" set.
~~~

#### 8.4.8 应用上下文参数

执行设备：K8S-M1

~~~shell
kubectl config use-context kubernetes \
  --kubeconfig=flanneld.kubeconfig
#输出
Switched to context "kubernetes"
~~~

#### 8.4.9 验证flanneld.kubeconfig

执行设备：K8S-M1

~~~shell
kubectl config view --kubeconfig=flanneld.kubeconfig
~~~

#### 8.4.10 分发flanneld.kubeconfig文件

执行设备：K8S-M1

~~~shell
cp flanneld.kubeconfig /opt/kubernetes/cfg
ansible node -m copy -a 'src=/opt/kubernetes/cfg/flanneld.kubeconfig dest=/opt/kubernetes/cfg'
~~~

#### 8.4.11 创建cidr的json文件

> 这两个文件的路径需要固定，不能更改，因为flannel的源码指定的这个目录

执行设备：K8S-M1

~~~shell
echo ${CLUSTER_CIDR}
cat <<EOF >/etc/kube-flannel/net-conf.json
{
  "Network": "${CLUSTER_CIDR}",
  "Backend": {
    "Type": "vxlan"
  }
}
EOF
ansible node -m copy -a 'src=/etc/kube-flannel/net-conf.json dest=/etc/kube-flannel/'
~~~

~~~shell
cat <<EOF >/etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
EOF
ansible node -m copy -a 'src=/etc/cni/net.d/10-flannel.conflist dest=/etc/cni/net.d/'
~~~

#### 8.4.12 创建flannel的systemd文件

> 因为一些变量不一样，需要每个节点分别执行
>
> 其中：NODE_NAME变量必须指定，且要和get node的hostname一致，且必须小写
>
> ExecStartPost的步骤，是为了生成/run/flannel/docker环境变量，这样后面docker加载该环境变量，就会改变docker0的IP地址

执行设备：node

~~~shell
export INTERFACE_NAME=ens33
cat <<EOF >/etc/systemd/system/flanneld.service
[Unit]
Description=Network fabric for containers
Documentation=https://github.com/coreos/flannel
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
Restart=always
RestartSec=5
KillMode=process
# This is needed because of this: https://github.com/coreos/flannel/issues/792
# Kubernetes knows the nodes by their FQDN so we have to use the FQDN
#Environment=NODE_NAME=my-node.foo.bar.com
# Note that we don't specify any etcd option. This is because we want to talk
# to the apiserver instead. The apiserver then talks to etcd on flannel's
# behalf.
Environment=NODE_NAME=$(echo "$HOSTNAME" | tr 'A-Z' 'a-z')
ExecStart=/opt/kubernetes/bin/flanneld \\
  --kube-subnet-mgr=true \\
  --kubeconfig-file=/opt/kubernetes/cfg/flanneld.kubeconfig \\
  --ip-masq=true \\
  --iface=${INTERFACE_NAME} \\
  --public-ip ${NODE_IP} \\
  --v=2
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker


[Install]
WantedBy=multi-user.target
EOF
~~~

#### 8.4.13 启动flanneld

执行设备：node

~~~shell
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
~~~

#### 8.4.14 配置Docker使用Flannel

> 我觉得可以先部署flanneld再部署docker，启动docker的时候，就可以修改好这些参数

执行设备：K8S-M1

~~~shell
# vim /lib/systemd/system/docker.service 

[Unit] #在Unit下面修改After和Requires后面增加flanneld.service
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service flanneld.service
Wants=network-online.target
Requires=docker.socket flanneld.service

[Service] #增加EnvironmentFile=-/run/flannel/docker  ExecStart 增加 $DOCKER_OPTS
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

~~~

~~~shell
#分发文件
ansible node -m copy -a 'src=/lib/systemd/system/docker.service dest=/lib/systemd/system/'
~~~

#### 8.4.15 重启docker

执行设备：node

~~~shell
systemctl daemon-reload
systemctl restart docker
~~~

#### 8.4.16 验证dockerIP

> 这个时候使用ifconfig的话，可以看到docker0 和flannel的网卡都是pod的cidr的地址了

执行设备：node任意

~~~
# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:cf:35:b5:f7  
          inet addr:100.68.3.1  Bcast:100.68.3.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

ens33     Link encap:Ethernet  HWaddr 00:0c:29:0d:e0:78  
          inet addr:192.168.137.167  Bcast:192.168.137.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:fe0d:e078/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:93591606 errors:0 dropped:0 overruns:0 frame:0
          TX packets:93020836 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:13552811439 (13.5 GB)  TX bytes:13463054173 (13.4 GB)

flannel.1 Link encap:Ethernet  HWaddr 56:2b:c5:78:64:dc  
          inet addr:100.68.3.0  Bcast:0.0.0.0  Mask:255.255.255.255
          inet6 addr: fe80::542b:c5ff:fe78:64dc/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:8 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:14179351 errors:0 dropped:0 overruns:0 frame:0
          TX packets:14179351 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:2642488310 (2.6 GB)  TX bytes:2642488310 (2.6 GB)
~~~

#### 参考资料

1. [馆长博客：不走etcd v2 api下二进制跑flannel的总结](https://zhangguanzhang.github.io/2019/03/15/flannel-bin/)
2. [馆长github文章：写给公司同事看的部署步骤 · zhangguanzhang/Kubernetes-ansible]([https://github.com/zhangguanzhang/Kubernetes-ansible/wiki/%E5%86%99%E7%BB%99%E5%85%AC%E5%8F%B8%E5%90%8C%E4%BA%8B%E7%9C%8B%E7%9A%84%E9%83%A8%E7%BD%B2%E6%AD%A5%E9%AA%A4](https://github.com/zhangguanzhang/Kubernetes-ansible/wiki/写给公司同事看的部署步骤))
3. [flannel配合docker使用-法兰绒/running.md掌握·coreos /法兰绒](https://github.com/coreos/flannel/blob/master/Documentation/running.md)
4. [flannel github官网](https://github.com/coreos/flannel)
5. [Kubernetes中的网络解析-以法兰绒为例·Kubernetes手册-Kubernetes中文指南/云原生应用架构实践手册，作者Jimmy Song（宋净超）](https://jimmysong.io/kubernetes-handbook/concepts/flannel.html)

## 9. 验证集群

### 9.1 查看节点

执行设备：K8S-M1

~~~shell
root@K8S-M1:~# kubectl get node -o wide 
NAME     STATUS   ROLES    AGE    VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-m1   Ready    <none>   2d     v1.18.4   192.168.137.166   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m2   Ready    <none>   2d     v1.18.4   192.168.137.167   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m3   Ready    <none>   2d     v1.18.4   192.168.137.168   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m4   Ready    <none>   2d     v1.18.4   192.168.137.169   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m5   Ready    <none>   2d     v1.18.4   192.168.137.170   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-n1   Ready    <none>   2d1h   v1.18.4   192.168.137.171   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
~~~

> node 的状态都变成了ready

### 9.2 测试pod

执行设备：K8S-M1

~~~shell
#测试yaml
cat <<EOF >/opt/kubernetes/deploy/test-ubuntu-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ubuntu-ds
  namespace: default
  labels:
    k8s-app: ubuntu-ds
spec:
  minReadySeconds: 10
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: ubuntu-ds
  template:
    metadata:
      labels:
        k8s-app: ubuntu-ds
    spec:
      affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: kubernetes.io/os
                  operator: In
                  values:
                  - linux
      containers:
      - name: ubuntu
        image: ubuntu:18.04
        imagePullPolicy: IfNotPresent
        command: ["/bin/bash","-c",  "sleep infinity"]
EOF
~~~

~~~shell
#应用yaml
cd /opt/kubernetes/deploy
kubectl apply -f test-ubuntu-ds.yaml
~~~

~~~shell
#查看ds
# kubectl get ds
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ubuntu-ds   6         6         6       6            6           <none>          123m
~~~

~~~shell
#查看pod
# kubectl get pod -o wide 
NAME              READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
ubuntu-ds-4bmfc   1/1     Running   0          2m54s   100.68.6.2   k8s-m5   <none>           <none>
ubuntu-ds-f722w   1/1     Running   0          2m54s   100.68.0.2   k8s-n1   <none>           <none>
ubuntu-ds-gkhz8   1/1     Running   0          2m54s   100.68.5.2   k8s-m3   <none>           <none>
ubuntu-ds-h6vmc   1/1     Running   0          2m54s   100.68.1.2   k8s-m1   <none>           <none>
ubuntu-ds-k946b   1/1     Running   0          2m54s   100.68.2.2   k8s-m4   <none>           <none>
ubuntu-ds-xz7cl   1/1     Running   0          2m54s   100.68.3.2   k8s-m2   <none>           <none>
~~~

> 可以看到上面的pod都获取到了IP

### 问题解决：

#### 1 无法查询pods日志问题

1. 问题描述：

   查询pod日志报如下信息：

   ~~~shell
   # kubectl logs ubuntu-ds-4bmfc
   error: You must be logged in to the server (the server has asked for the client to provide credentials ( pods/log ubuntu-ds-4bmfc))
   ~~~

2. 问题解决：

   1. 创建rabc文件：

      执行设备： K8S-M1

      ~~~shell
      cd /opt/kubernetes/deploy
      cat <<EOF  >apiserver-to-kubelet-rbac.yml 
      kind: ClusterRoleBinding
      apiVersion: rbac.authorization.k8s.io/v1
      metadata:
        name: kubelet-api-admin
      subjects:
      - kind: User
        name: kubernetes
        apiGroup: rbac.authorization.k8s.io
      roleRef:
        kind: ClusterRole
        name: system:kubelet-api-admin
        apiGroup: rbac.authorization.k8s.io
      EOF
      ~~~

   2. 应用
      执行设备： K8S-M1

      ~~~shell
      # 应用
      kubectl apply -f apiserver-to-kubelet-rbac.yml
      ~~~

   3. 验证

      执行设备： K8S-M1

      ~~~shell
      # kubectl logs ubuntu-ds-4bmfc
      Error from server: Get https://k8s-m5:10250/containerLogs/default/ubuntu-ds-4bmfc/ubuntu: dial tcp: lookup k8s-m5 on 223.5.5.5:53: no such host
      ~~~

   4. 参考文档：

      1. [Kubernetes 无法查看 pods 日志问题]([https://www.yp14.cn/2020/05/13/Kubernetes-%E6%97%A0%E6%B3%95%E6%9F%A5%E7%9C%8B-pods-%E6%97%A5%E5%BF%97%E9%97%AE%E9%A2%98/](https://www.yp14.cn/2020/05/13/Kubernetes-无法查看-pods-日志问题/))
      2. [Kubernetes v1.18.2 二进制高可用部署](https://www.jianshu.com/p/ddd7a73d7b2a)
      3. [kubelet官方认证文档](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/)

## 10 addon部署

### 10.1  集群CoreDNS部署

> coredns为集群内部提供dns服务

执行设备： K8S-M1

1. 下载 CoreDNS 项目

   > `deploy.sh` 是一个便捷的脚本，用于生成coredns yaml 配置

~~~shell
# 安装依赖 jq 命令
apt  install jq -y

cd /home/xxsun5/pkg
# 下载 CoreDNS 项目
 git clone https://github.com/coredns/deployment.git
 cd deployment/kubernetes
~~~

2. 修改`CLUSTER_DNS_IP` 

   > 默认情况下 `CLUSTER_DNS_IP` 是自动获取kube-dns的集群ip的，但是由于没有部署kube-dns所以只能手动指定一个集群ip。

   ~~~shell
   vim deploy.sh
   if [[ -z $CLUSTER_DNS_IP ]]; then
     # Default IP to kube-dns IP
     #CLUSTER_DNS_IP=$(kubectl get service --namespace kube-system kube-dns -o jsonpath="{.spec.clusterIP}")
     CLUSTER_DNS_IP=100.67.0.2
   ~~~

3. 部署

   ~~~shell
   # 查看执行效果，并未开始部署
   $ ./deploy.sh
   
   # 执行部署
   $ ./deploy.sh | kubectl apply -f -
   
   # 查看 Coredns
   $ kubectl get svc,pods -n kube-system| grep coredns
   ~~~

4. 验证 Coredns 解析

   ~~~shell
   cd /opt/kubernetes/deploy
   cat <<EOF >busybox.yaml 
   apiVersion: v1
   kind: Pod
   metadata:
     name: busybox
     namespace: default
   spec:
     containers:
     - name: busybox
       image: busybox:1.28.4
       command:
         - sleep
         - "3600"
       imagePullPolicy: IfNotPresent
     restartPolicy: Always
   EOF
   
   ~~~

   ~~~shell
   # 部署
    kubectl apply -f busybox.yaml
   
   # 测试解析，下面是解析正常
   # kubectl exec -i busybox -n default -- nslookup kubernetes
   Server:    100.67.0.2
   Address 1: 100.67.0.2 kube-dns.kube-system.svc.cluster.local
   
   Name:      kubernetes
   Address 1: 100.67.0.1 kubernetes.default.svc.cluster.local
   ~~~

   

### 10.2 部署集群监控服务 Metrics Server

> metrics-server是kubectl top命令信息来源提供。

执行设备： K8S-M1

1. 拉取 v0.3.6 版本

   ~~~shell
   cd /home/xxsun5/pkg
   git clone https://github.com/kubernetes-sigs/metrics-server.git -b v0.3.6
   ~~~

2. 修改文件

   > 主要增加了资源限制和command的两个参数,
   >
   > 除此之外，metrics的镜像是谷歌的一个镜像： k8s.gcr.io/metrics-server-amd64:v0.3.6，这个镜像默认pull不下来，可以改成下面的镜像，也可以pull下面的镜像，然后docker tag改名
   >
   > ```bash
   > yangpeng2468/metrics-server-amd64:v0.3.6
   > ```

   ~~~shell
   docker pull yangpeng2468/metrics-server-amd64:v0.3.6
   docker tag yangpeng2468/metrics-server-amd64:v0.3.6 k8s.gcr.io/metrics-server-amd64:v0.3.6
   ~~~

   

   ~~~shell
   cd metrics-server/deploy/1.8+
   cp metrics-server-deployment.yaml metrics-server-deployment.yaml.bak
   ##修改对比如下：
   diff metrics-server-deployment.yaml metrics-server-deployment.yaml.bak 
   33,44c33
   <         imagePullPolicy: IfNotPresent
   <         resources:
   <           limits:
   <             cpu: 400m
   <             memory: 1024Mi
   <           requests:
   <             cpu: 50m
   <             memory: 50Mi
   <         command:
   <         - /metrics-server
   <         - --kubelet-insecure-tls
   <         - --kubelet-preferred-address-types=InternalIP
   ---
   >         imagePullPolicy: Always
   ~~~

3. 部署

   ~~~shell
   kubectl apply -f .
   ~~~

4. 验证

   > ```bash
   > # 内存单位 Mi=1024*1024字节  M=1000*1000字节
   > # CPU单位 1核=1000m 即 250m=1/4核
   > ```

   ~~~shell
   # kubectl get pod -n kube-system |grep metrics
   metrics-server-668f6bb997-9mrnm   1/1     Running   0          3h16m
   # kubectl top node
   NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
   k8s-m1   275m         13%    2180Mi          73%       
   k8s-m2   251m         12%    2453Mi          82%       
   k8s-m3   234m         11%    2556Mi          85%       
   k8s-m4   221m         11%    2590Mi          86%       
   k8s-m5   279m         13%    2747Mi          92%       
   k8s-n1   63m          3%     1174Mi          39%      
   # kubectl top pod
   NAME              CPU(cores)   MEMORY(bytes)   
   busybox           0m           0Mi             
   ubuntu-ds-4bmfc   0m           1Mi             
   ubuntu-ds-f722w   0m           0Mi             
   ubuntu-ds-gkhz8   0m           1Mi             
   ubuntu-ds-h6vmc   0m           1Mi             
   ubuntu-ds-k946b   0m           1Mi             
   ubuntu-ds-xz7cl   0m           1Mi   
   
   ~~~

### 10.3 Kuernetes Dashboard 2.0 部署

> 需要K8S集群部署 metrics-server，这样才能正常查看 Dashboard 监控指标.

#### 10.3.1 Ingress Nginx 部署

执行设备： K8S-M1

1. 创建目录

~~~shell
mkdir /opt/kubernetes/deploy
mkdir /opt/kubernetes/deploy/ingress_nginx
cd /opt/kubernetes/deploy/ingress_nginx
~~~

2. 准备部署的yaml文件

~~~yaml
vim ingress-controller-deploy.yaml
i
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  # 客户端请求头的缓冲区大小 
  client-header-buffer-size: "512k"
  # 设置用于读取大型客户端请求标头的最大值number和size缓冲区
  large-client-header-buffers: "4 512k"
  # 读取客户端请求body的缓冲区大小
  client-body-buffer-size: "128k"
  # 代理缓冲区大小
  proxy-buffer-size: "256k"
  # 代理body大小
  proxy-body-size: "50m"
  # 服务器名称哈希大小
  server-name-hash-bucket-size: "128"
  # map哈希大小
  map-hash-bucket-size: "128"
  # SSL加密套件
  ssl-ciphers: "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA"
  # ssl 协议
  ssl-protocols: "TLSv1 TLSv1.1 TLSv1.2"
  # 日志格式
  log-format-upstream: '[$the_real_ip] - $remote_user [$time_local] "$request" $status $body_bytes_sent $request_time "$http_referer" $host DIRECT/$upstream_addr $upstream_http_content_type "$http_user_agent" "$http_x_forwarded_for" $request_length [$proxy_upstream_name] $upstream_response_length $upstream_response_time $upstream_status $req_id'

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      nodePort: 30443
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.24.1
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              cpu: 1000m
              memory: 1000Mi
            requests:
              cpu: 30m
              memory: 50Mi
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
~~~

3. 配置支持 Socket.io

> 这个其实是演示nginx-ingress的yaml
>
> 重点是使用nginx $http_x_forwarded_for做客户端ip地址一致hash，实现会话保持

~~~yaml
cat <<EOF >ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: websocket
  namespace: default
  annotations:
    # 代理发送超时
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    # 代理读取超时
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    # 代理连接超时
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "3600"
    # 基于客户端出口ip哈希
    nginx.ingress.kubernetes.io/upstream-hash-by: "\$http_x_forwarded_for"
spec:
  rules:
  - host: websocket.example.com
    http:
      paths:
      - backend:
          serviceName: websocket
          servicePort: 8080
        path: /
EOF
~~~

4. 应用yaml

~~~shell
kubectl apply -f ingress-controller-deploy.yaml 
kubectl apply -f ingress.yaml
~~~

5. 查看

~~~shell
# kubectl -n ingress-nginx get svc
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   100.67.178.73   <none>        80:30080/TCP,443:30443/TCP   16m
root@K8S-M1:/opt/kubernetes/deploy/ingress_nginx# kubectl -n ingress-nginx get pod
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-cbfd8945-cwbj2   1/1     Running   0          5h1m
nginx-ingress-controller-cbfd8945-hndmq   1/1     Running   0          5h1m
~~~

6. 参考文档
   1. [K8S ingress-nginx部署]([https://www.yp14.cn/2019/11/19/K8s-Ingress-Nginx-%E6%94%AF%E6%8C%81-Socket-io/](https://www.yp14.cn/2019/11/19/K8s-Ingress-Nginx-支持-Socket-io/))
   2. [张磊 ingress讲解（公司内部文档，公司以外看不到）]([http://wiki.iflytek.com/pages/viewpage.action?pageId=265834176#/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Kubernetes%2F39%20%E8%B0%88%E8%B0%88Service%E4%B8%8EIngress.md?branch=master](http://wiki.iflytek.com/pages/viewpage.action?pageId=265834176#/深入剖析Kubernetes%2F39 谈谈Service与Ingress.md?branch=master))

#### 10.3.2 自定义证书

1. 生成私钥，需要设置密码

   ~~~shell
   mkdir /home/xxsun5/pkg/domain_ssl
   cd /home/xxsun5/pkg/domain_ssl
   openssl genrsa -des3 -out xxsun5.key 2048
   ~~~

2. 生成CA证书，需要输入密码

   ~~~shell
   # 生成CA证书，需要输入密码
   openssl req -sha512 -new \
       -subj "/C=CN/ST=JS/L=WX/O=zwx/OU=xxsun5/CN=xxsun5.cn" \
       -key xxsun5.key \
       -out xxsun5.csr
   ~~~

3. 备份证书

   ~~~shell
   cp xxsun5.key xxsun5.key.org
   ~~~

4. 退掉私钥密码，以便docker访问（也可以参考官方进行双向认证)

   ~~~shell
   openssl rsa -in xxsun5.key.org -out xxsun5.key
   ~~~

5. 使用证书进行签名

   ~~~shell
   openssl x509 -req -days 365 -in xxsun5.csr -signkey xxsun5.key -out xxsun5.crt
   ~~~

   ~~~shell
   # ls
   xxsun5.crt  xxsun5.csr  xxsun5.key  xxsun5.key.org
   ~~~

6. 分发证书

   ~~~shell
   cp xxsun5.crt xxsun5.key /opt/kubernetes/ssl/
   ~~~

7. 参考文档：

   1. [Harbor搭建文档（内有自签https证书步骤）](https://www.cnblogs.com/zhangmingcheng/p/12753959.html)

#### 10.3.3 Dashboard部署

1. 创建 kubernetes-dashboard-certs secret

   > 注意：自定义证书 kubernetes-dashboard-certs secret 必须存储在与Kubernetes仪表板相同的 Namespaces。

   ~~~shell
   kubectl create ns kubernetes-dashboard
   kubectl create secret generic kubernetes-dashboard-certs --from-file=/opt/kubernetes/ssl/ -n kubernetes-dashboard
   ~~~

2. 下载 dashboard yaml 文件

   > 这个yaml似乎需要梯子，需要通过代理拿到

   ~~~shell
   mkdir /opt/kubernetes/deploy/dashborad/
   cd /opt/kubernetes/deploy/dashborad/
   wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
   ~~~

3. 修改 dashboard yaml 文件

   ~~~shell
   vim recommended.yaml
   
   # 把创建 kubernetes-dashboard-certs Secret 注释掉，前面已通过命令创建。
   
   #apiVersion: v1
   #kind: Secret
   #metadata:
   #  labels:
   #    k8s-app: kubernetes-dashboard
   #  name: kubernetes-dashboard-certs
   #  namespace: kubernetes-dashboard
   #type: Opaque
   #下面是修改deployment的args字段
   # 添加ssl证书路径，关闭自动更新证书，添加多长时间登出。
   # 注意：--tls-key-file --tls-cert-file 引用名称，要与上面创建 kubernetes-dashboard-certs Secret 引用的证书文件名称一样。
             args:
               #- --auto-generate-certificates
               - --namespace=kubernetes-dashboard
               - --tls-key-file=xxsun5.key
               - --tls-cert-file=xxsun5.crt
               - --token-ttl=3600
   ~~~

4. 部署 dashboard

   ~~~shell
   kubectl  apply -f recommended.yaml
   ~~~

5. 查看 dashboard

   ~~~shell
   kubectl  get pods -n kubernetes-dashboard
   NAME                                         READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
   dashboard-metrics-scraper-6b4884c9d5-kzz9s   1/1     Running   0          64s   100.68.6.3   k8s-m5   <none>           <none>
   kubernetes-dashboard-859fc85b4-l8tdk         1/1     Running   0          64s   100.68.2.3   k8s-m4   <none>           <none>
   ~~~

6. 创建登陆用户-admin-user 管理员 yaml 配置

   ~~~yaml
   cat <<EOF > create-admin.yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: admin-user
     namespace: kubernetes-dashboard
   
   ---
   
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: admin-user
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
   - kind: ServiceAccount
     name: admin-user
     namespace: kubernetes-dashboard
   EOF
   ~~~

7. 创建登陆用户

   ~~~shell
   kubectl apply -f create-admin.yaml
   ~~~

8. 创建dashboard-ingress 的 tls证书

   ~~~shell
   kubectl create secret tls k8s-dashboard --key /opt/kubernetes/ssl/xxsun5.key --cert /opt/kubernetes/ssl/xxsun5.crt -n kubernetes-dashboard
   ~~~

9. 配置 dashboard ngress Nginx 提供访问入口

   ~~~yaml
   cat <<EOF > k8s-dashboard-ingress.yaml
   apiVersion: networking.k8s.io/v1beta1
   kind: Ingress
   metadata:
     name: k8s-dashboard-ingress
     namespace: kubernetes-dashboard
     annotations:
       kubernetes.io/ingress.class: "nginx"
       # 开启use-regex，启用path的正则匹配
       nginx.ingress.kubernetes.io/use-regex: "true"
       nginx.ingress.kubernetes.io/rewrite-target: /
       # 默认为 true，启用 TLS 时，http请求会 308 重定向到https
       nginx.ingress.kubernetes.io/ssl-redirect: "true"
       # 默认为 http，开启后端服务使用 proxy_pass https://协议
       nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
   spec:
     rules:
     - host: kubernetes-dashboard.xxsun5.cn
       http:
         paths:
         - path: /
           backend:
             serviceName: kubernetes-dashboard
             servicePort: 443
     tls:
     - secretName: k8s-dashboard
       hosts:
       - xxsun5.cn
   EOF
   ~~~

10. 创建

    ~~~shell
    kubectl apply -f k8s-dashboard-ingress.yaml
    ~~~

11. 查看登陆 token

    ~~~shell
    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
    ~~~

12. 访问验证

    > 前提是访问dashboard界面的系统已经了hosts解析，如下
    > `192.168.137.166 kubernetes-dashboard.xxsun5.cn`

    访问的地址为：

    ~~~shell
    https://kubernetes-dashboard.xxsun5.cn:30443/#/login
    ~~~

    ![image.png](https://i.loli.net/2020/07/03/p9CvLPkAtBJZb3m.png)

    > 用上面的token访问

    ![image.png](https://i.loli.net/2020/07/03/BFiIKejT2OYZJGX.png)

    ![image.png](https://i.loli.net/2020/07/03/7jPtLlXfFk4UHTM.png)

    

13. 参考文档：

    1. [K8S Dashboard 2.0 部署并使用 ]([https://www.yp14.cn/2020/05/16/K8S-Dashboard-2-0-%E9%83%A8%E7%BD%B2%E5%B9%B6%E4%BD%BF%E7%94%A8-Ingress-Nginx-%E6%8F%90%E4%BE%9B%E8%AE%BF%E9%97%AE%E5%85%A5%E5%8F%A3/](https://www.yp14.cn/2020/05/16/K8S-Dashboard-2-0-部署并使用-Ingress-Nginx-提供访问入口/))

## 参考文档：

1. [v1.18 发布说明 | Kubernetes](https://kubernetes.io/zh/docs/setup/release/notes/)
2. [k8s用户安全管理模型简析](https://mp.weixin.qq.com/s/rYj0H3PeQ4xAg-JOOLDVXA)
