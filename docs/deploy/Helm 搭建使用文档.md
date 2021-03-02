# Helm 部署使用

## 前言：

> helm 类似于Linux系统下的包管理器，如yum/apt等，可以方便快捷的将之前打包好的yaml文件快速部署进kubernetes内，方便管理维护.

### 基础概念：

- helm：一个命令行下客户端工具，主要用于kubernetes应用chart的创建/打包/发布已经创建和管理和远程Chart仓库。

- Tiller：helm的服务端，部署于kubernetes内，Tiller接受helm的请求，并根据chart生成kubernetes部署文件（helm称为release），然后提交给 Kubernetes 创建应用。Tiller 还提供了 Release 的升级、删除、回滚等一系列功能。**这个角色到版本3之后就没有了**

- Chart： helm的软件包，采用tar格式，其中包含运行一个应用所需的所有镜像/依赖/资源定义等，还可能包含kubernetes集群中服务定义

- Release：在kubernetes中集群中运行的一个Chart实例，在同一个集群上，一个Chart可以安装多次，每次安装均会生成一个新的release。

- Repository：用于发布和存储Chart的仓库。阿里的helm源地址为：

  ```
  https://developer.aliyun.com/hub
  ```

  ### helm 架构图示

  ![image.png](https://i.loli.net/2020/12/09/rZ1MlNH5IYXKq4S.png)

  参考文档：[玩K8S不得不会的HELM](https://www.imooc.com/article/291355)
  

## 版本选择：

安装之前最好去官方看下版本所支持和对接的K8S版本，以防止使用的时候有什么不适应的情况。

版本对应地址：[版本支持查看入口](https://helm.sh/docs/topics/version_skew/) 当前版本对应如下：

~~~shell
Helm Version	Supported Kubernetes Versions
3.4.x                 1.19.x - 1.16.x
3.3.x                 1.18.x - 1.15.x
3.2.x                 1.18.x - 1.15.x
3.1.x                 1.17.x - 1.14.x
3.0.x                 1.16.x - 1.13.x
2.16.x	              1.16.x - 1.15.x
2.15.x	              1.15.x - 1.14.x
2.14.x	              1.14.x - 1.13.x
2.13.x	              1.13.x - 1.12.x
2.12.x	              1.12.x - 1.11.x
2.11.x	              1.11.x - 1.10.x
2.10.x	              1.10.x - 1.9.x
2.9.x	              1.10.x - 1.9.x
2.8.x	              1.9.x - 1.8.x
2.7.x	              1.8.x - 1.7.x
2.6.x	              1.7.x - 1.6.x
2.5.x	              1.6.x - 1.5.x
2.4.x	              1.6.x - 1.5.x
2.3.x	              1.5.x - 1.4.x
2.2.x	              1.5.x - 1.4.x
2.1.x	              1.5.x - 1.4.x
2.0.x	              1.4.x - 1.3.x
~~~

**版本2和版本3部署差别很大，版本3部署起来是真的方便**

## 版本3安装：

我自己测试环境的K8S集群版本如下：

~~~shell
root@K8S-M1:~/deploy# kubectl version --short
Client Version: v1.18.4
Server Version: v1.18.4
root@K8S-M1:~/deploy# kubectl get node -o wide 
NAME     STATUS   ROLES    AGE    VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-m1   Ready    master   151d   v1.18.4   192.168.137.166   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m2   Ready    master   151d   v1.18.4   192.168.137.167   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m3   Ready    master   151d   v1.18.4   192.168.137.168   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m4   Ready    master   151d   v1.18.4   192.168.137.169   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-m5   Ready    master   151d   v1.18.4   192.168.137.170   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
k8s-n1   Ready    worker   151d   v1.18.4   192.168.137.171   <none>        Ubuntu 16.04.4 LTS   4.4.0-116-generic   docker://19.3.6
~~~

所以安装了最新版：

![image.png](https://i.loli.net/2020/12/09/odnAysJrp4iBtfC.png)



###  下载包：

~~~shell
wget https://get.helm.sh/helm-v3.4.1-linux-amd64.tar.gz
~~~

### 解压包

~~~shell
tar -zxvf helm-v3.4.1-linux-amd64.tar.gz
~~~

### 将二进制文件移到环境变量路径下

~~~shell
mv linux-amd64/helm /usr/local/bin/helm
~~~

### 添加源

> 添加阿里源

~~~shell
helm repo add apphub https://apphub.aliyuncs.com
~~~

> 查看添加的源：

~~~shell
# helm repo list
NAME  	URL                        
apphub	https://apphub.aliyuncs.com
~~~

### 验证安装nginx

#### 查看repo里的软件版本：

~~~shell
# helm search repo apphub/nginx
NAME                           	CHART VERSION	APP VERSION         	DESCRIPTION                                       
apphub/nginx                   	5.1.5        	1.16.1              	Chart for the nginx server                        
apphub/nginx-ingress           	1.30.3       	0.28.0              	An nginx Ingress controller that uses ConfigMap...
apphub/nginx-ingress-controller	5.3.4        	0.29.0              	Chart for the nginx Ingress controller            
apphub/nginx-lego              	0.3.1        	                    	Chart for nginx-ingress-controller and kube-lego  
apphub/nginx-php               	1.0.0        	nginx-1.10.3_php-7.0	Chart for the nginx php server 
~~~

~~~shell
#helm show chart apphub/nginx
apiVersion: v1
appVersion: 1.16.1
description: Chart for the nginx server
home: http://www.nginx.org
icon: https://bitnami.com/assets/stacks/nginx/img/nginx-stack-220x234.png
keywords:
- nginx
- http
- web
- www
- reverse proxy
maintainers:
- email: containers@bitnami.com
  name: Bitnami
name: nginx
sources:
- https://github.com/bitnami/bitnami-docker-nginx
version: 5.1.5
~~~

#### 安装

~~~shell
# helm install apphub/nginx --generate-name
NAME: nginx-1607483942
LAST DEPLOYED: Wed Dec  9 11:19:04 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the NGINX URL:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w nginx-1607483942'

  export SERVICE_IP=$(kubectl get svc --namespace default nginx-1607483942 --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
  echo "NGINX URL: http://$SERVICE_IP/"
~~~

#### 查看

~~~shell
# helm ls
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
nginx-1607483942	default  	1       	2020-12-09 11:19:04.385008546 +0800 CST	deployed	nginx-5.1.5	1.16.1     
~~~

~~~shell
# kubectl get pod  |grep nginx
nginx-1607483942-67c48b7866-d699f   1/1     Running   0          2m48s
~~~

~~~shell
# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-1607483942-67c48b7866   1         1         1       6m30s
~~~

~~~shell
# kubectl get svc
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
nginx-1607483942   LoadBalancer   100.67.132.0   <pending>     80:31989/TCP,443:30268/TCP   80s
~~~

#### 访问验证：

~~~shell
# curl http://100.67.132.0
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

#### 删除

~~~shell
# helm delete nginx-1607483942
release "nginx-1607483942" uninstalled
~~~

#### 参考文档

- 1. [helm官方安装文档](https://helm.sh/docs/intro/install/)
- 2. [helm GitHUb地址](https://github.com/helm/helm)
- 3. [玩K8S不得不会的HELM](https://www.imooc.com/article/291355)

## 版本2安装

版本2安装起来比版本3要麻烦得多，需要有rbac鉴权、需要安装tiller。不过这个集群的版本太低，版本如下：

~~~shell
# kubectl version --short
Client Version: v1.10.11
Server Version: v1.10.11
~~~

对照上面的版本选择的参考应该安装helm的版本为：`2.11.X`。这次安装的时候也确实是先按照版本：`2.11.0`安装的。**但是安装tiller的时候，下载的镜像怎么都running不起来，最后换了2.14.2的镜像一下就running了**。

 ### 下载

