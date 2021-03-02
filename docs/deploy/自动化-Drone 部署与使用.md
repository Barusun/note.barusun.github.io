#  Drone 部署与使用

## 自动化流程图例

<img src="https://i.loli.net/2021/01/05/B7hHIjyQga5P3wE.png" alt="image.png" style="zoom:150%;" />

## Drone架构

![image.png](https://i.loli.net/2021/01/11/thc7grOVk61a8P5.png)

​		实际部署的时候需要部署：Drone Server和Drone Runner

- Drone Server : 是平时操作的持续化集成平台，可以根据不同的种类的仓库，创建不同的server

  ![image.png](https://i.loli.net/2021/01/11/YirqRjZLmzhPgdx.png)

- Drone Runner:  是执行任务的发起者；可以根据不同的环境，创建不同种类的runner

  ![image.png](https://i.loli.net/2021/01/11/VRxHetZlYGKcNaQ.png)

- Drone Pipleline： 任务的最终执行者；生命周期是一次性的，自动化任务开始，pipleline创建，任务结束，pipleline销毁。

  

## 前言

> 其实Drone的部署很简单，对着[官网文档](https://docs.drone.io/)其实就差不多了。只不过从懵开始，对整个流程存在很多疑问，所以还是会走些弯路，费些时间。除了搭建起来，使用拉通整个流程也需要对上面流程图的每个步骤有点基础。

关于部署方式，可以从`Kubernetes`部署；也可以`docker`部署，官网给的参考是`docekr`部署的，本次也是按照docker部署。最后部署完成会附上`Kubernetes`的`yaml`文档，以供`Kubernetes`部署的时候参考。 

## 环境说明

> `kubernetes`部署得需要`kubernetes`环境；`docker`部署得需要`docker`环境。

环境说明:

~~~shell
#系统信息：
# cat /etc/issue
Ubuntu 16.04.6 LTS \n \l
#docker信息
# docker version
Client:
 Version:           18.09.6
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        481bc77
 Built:             Sat May  4 02:35:27 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.6
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.8
  Git commit:       481bc77
  Built:            Sat May  4 01:59:36 2019
  OS/Arch:          linux/amd64
  Experimental:     false
~~~

## Drone Server部署

### Gitlab 准备

> 部署的时候需要gitlab的应用的两个认证信息：`DRONE_GITLAB_CLIENT_ID`(gitlab的客户端id)和`DRONE_GITLAB_CLIENT_SECRET`(gitlab的客户端密钥)。

生成这两个配置如下图所示：

![image.png](https://i.loli.net/2021/01/11/skhc5M2G3uHXriQ.png)

![image.png](https://i.loli.net/2021/01/11/oGKpadS3zMfsiuT.png)
![image.png](https://i.loli.net/2021/01/11/hjmsOyu2Mbao1QT.png)

​		官网也有生成步骤，我这里不过是赘述了一遍。

参考文档：[官网：Drone使用gitlab来实现自动化](https://docs.drone.io/server/provider/gitlab/)

### RPC_SECRET创建

​		该密钥用于Drone_runner和Droneserver通信认证，用以下命令生成

~~~shell
# openssl rand -hex 16
738d5cacdb95d6af87838179c33af063
~~~

	### docker 镜像准备 

> 下载好也行，不下载好，启动的时候本地没有，也会去远端拉

~~~shell
docker pull drone/drone:1
~~~

### 启动 Drone Server

> 配置以下内容，启动drone server

~~~shell
# cat drone_start.sh 
#/bin/bash
export DRONE_GITLAB_CLIENT_ID='8a7xxxxx54ce2239207a91546145a704050185263f1d46192363f4e6809a9'
export DRONE_GITLAB_CLIENT_SECRET='dfcxxxxxxad4cae4fde769887920a63399ce106c23af8d868a97f99'
export DRONE_RPC_SECRET='e3571d0595dfbc59decf00a5baeb0486'
export DRONE_SERVER_HOST='172.31.223.247:10080'
export DRONE_SERVER_PROTO='http'
export DRONE_GITLAB_SERVER='https://git.xxxx.com'
export DRONE_AGENTS_ENABLED='true'
export DRONE_USER_CREATE='username:user1,username:user2,admin:true'
docker run \
  --volume=/data/deploy/drone/data:/data \
  --env=DRONE_AGENTS_ENABLED=${DRONE_AGENTS_ENABLED} \
  --env=DRONE_GITLAB_SERVER=${DRONE_GITLAB_SERVER} \
  --env=DRONE_GITLAB_CLIENT_ID=${DRONE_GITLAB_CLIENT_ID} \
  --env=DRONE_GITLAB_CLIENT_SECRET=${DRONE_GITLAB_CLIENT_SECRET} \
  --env=DRONE_RPC_SECRET=${DRONE_RPC_SECRET} \
  --env=DRONE_SERVER_HOST=${DRONE_SERVER_HOST} \
  --env=DRONE_SERVER_PROTO=${DRONE_SERVER_PROTO} \
  --env=DRONE_USER_CREATE=${DRONE_USER_CREATE} \
  --publish=10080:80 \
  --publish=10443:443 \
  --restart=always \
  --detach=true \
  --name=drone \
  drone/drone:1

~~~

~~~shell
bash drone_start.sh
##检查
# docker ps |grep drone
7c5bd178559c        drone/drone:1          "/bin/drone-server"      2 weeks ago         Up 2 weeks          0.0.0.0:10080->80/tcp, 0.0.0.0:10443->443/tcp   drone
# ss -ntlp|grep 10080
LISTEN     0      128         :::10080                   :::*                   users:(("docker-proxy",pid=4093,fd=4))
~~~

​		现在回过头来看gitlab和drone的联动，流程大概是这样的：第一次访问drone的时候，会自动跳转至gitlab的登陆界面，图一。登陆的这个仓库的目标地址是在drone启动的时候Drone_指定的（如上配置`DRONE_GITLAB_SERVER`）。登陆gitlab，会弹出如下图二，问你是否选择接受认证。接受认证之后，gitlab回自动跳转回drone界面（生成Gitlab的认证的时候填的目标drone_server地址)，图三。第一次登陆也会在这时候会让你选你想要自动化的仓库。



![image.png](https://i.loli.net/2021/01/11/mxPAvORThLg4I6y.png)

​																            图 一

![image.png](https://i.loli.net/2021/01/11/ZlKzNwHWpm4Tuxc.png)

​																            图 二

![image.png](https://i.loli.net/2021/01/11/sVehLUJAHymKGfN.png)

​																            图 三

两个细节：

1. 选择接受认证，则在gitlab界面右上角头像—>setting—>applications的下面会看到一个认证过的信息，不想使用这个认证的话，可以在这里删掉。如下图：

   ![image.png](https://i.loli.net/2021/01/11/sfJ5BIrD9lCKbtd.png)

2. 启动drone server的时候，如果这个配置：`DRONE_USER_CREATE`，没有创建管理员用户的话，那么非管理员用户登陆使用drone，ACTIVE 仓库（图一）的时候，会没有以下选项（图二），如果不是仓库管理员，会出现图三情况：

   ![image.png](https://i.loli.net/2021/01/11/RLaYZbdDvjVcsxn.png)

   ​																            图 一：激活仓库

   ![image.png](https://i.loli.net/2021/01/11/8o3ht9LWVbXDkr5.png)

​																            图 二： 不是Drone管理员没有该选项

![image.png](https://i.loli.net/2021/01/11/dQGqE8VoLyvRYj2.png)

​																            图 三：不是仓库管理员

### 附：K8S部署yaml

~~~yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: drone-server-ds
  namespace: drone
  labels:
    k8s-app: drone-server
spec:
  minReadySeconds: 2
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: drone-server
  template:
    metadata:
      labels:
        k8s-app: drone-server
    spec:
      nodeSelector:
        drone: "true"
      volumes:
      - name: timezone
        hostPath:
          path: /etc/localtime
      - name: data
        hostPath:
          path: /data/drone/data
      containers:
      - name: drone-server # container name
        image: drone/drone:1.10.0
        imagePullPolicy: IfNotPresent
        readinessProbe:
          tcpSocket:
            port: 80
            #服务启动后，多长时间服务可用
          initialDelaySeconds: 3
          periodSeconds: 2
        env:
        - name: DRONE_AGENTS_ENABLED
          value: "true"
        - name: DRONE_GITLAB_SERVER
          value: https://git.xxxxx.com
        # 下面环境变量值从 gitlab 的 Drone application 拷贝过来
        - name: DRONE_GITLAB_CLIENT_ID
          value: xxxxx
        # 下面环境变量值从 gitlab 的 Drone application 拷贝过来
        - name: DRONE_GITLAB_CLIENT_SECRET
          value: xxxx
        - name: DRONE_RPC_SECRET
          value: b68cf14a9b416126122072d5f4d1132b
        - name: DRONE_SERVER_HOST
          value: 172.xx.2xx.xx
        - name: DRONE_SERVER_PROTO
          value: http
        - name: DRONE_USER_CREATE
          value: username:xxxx,admin:true
        - name: LANG
          value: C.UTF-8
        - name: NODE
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        volumeMounts:
        - name: timezone
          mountPath: /etc/localtime
        - name: data
          mountPath: /data
        ports:
        - name: profiler
          containerPort: 2021
          hostPort: 12021
          protocol: TCP   
        - name: http
          containerPort: 80
          hostPort: 80
          protocol: TCP
        - name: https
          containerPort: 443
          hostPort: 443
          protocol: TCP
        #Limited resources for CPU
        resources:
          requests:
            cpu: "200m"
            memory: 100Mi
          limits:
            cpu: "2000m"
            memory: 1000Mi
~~~

## Drone Runner 部署

> dorne runner如上述可以有很多类型，需要注意的是:`.drone.yml`中的type字段要和你的runner的类型要一致。不然的话整个自动化流程也拉不同。这边的运行环境是Kubernetes，所以部署type为Kubernetes 的Drone runner。

### Runner 鉴权

> drone runner这个角色，需要动态的创建临时pipeline的任务pod，需要与集群通信，创建、删除pod等权限，以下为授权的yaml，将drone runner 这个角色的权限赋给default 这个账户

~~~yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: drone
  name: drone-runner
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - create
  - delete
- apiGroups:
  - ""
  resources:
  - pods
  - pods/log
  verbs:
  - get
  - create
  - delete
  - list
  - watch
  - update

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: drone-runner
  namespace: drone
subjects:
- kind: ServiceAccount
  name: default
  namespace: drone
roleRef:
  kind: Role
  name: drone-runner
  apiGroup: rbac.authorization.k8s.io
~~~

 ~~~shell
#应用
kubectl apply -f role-runner.yaml 
#查看
# kubectl -n drone get ServiceAccount
NAME      SECRETS   AGE
default   1         23d
# kubectl -n drone get role
NAME           AGE
drone-runner   22d
# kubectl -n drone get RoleBinding
NAME           AGE
drone-runner   22d
 ~~~

### runner启动

~~~shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drone
  namespace: drone
  labels:
    app.kubernetes.io/name: drone
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: drone
  template:
    metadata:
      labels:
        app.kubernetes.io/name: drone
    spec:
      containers:
      - name: runner
        image: drone/drone-runner-kube:linux-amd64
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
        env:
        - name: DRONE_RPC_HOST
          value: 172.31.223.247:10080
        - name: DRONE_RPC_PROTO
          value: http
        - name: DRONE_RPC_SECRET
          value: e3571xxxxx59decf00a5baeb0486
        - name: DRONE_NAMESPACE_DEFAULT
          value: "drone"
      nodeSelector:
        drone-runner: "true"
      restartPolicy: Always
~~~

~~~shell
#应用
kubectl apply -f role-runner.yaml
# kubectl -n drone get pod -o wide -l app.kubernetes.io/name=drone
NAME                     READY   STATUS    RESTARTS   AGE   IP             NODE             NOMINATED NODE   READINESS GATES
drone-84fb76fbd4-8w8kt   1/1     Running   0          22d   10.244.4.233   hfa-xxx-131186   <none>           <none>
# kubectl -n drone logs drone-84fb76fbd4-8w8kt
time="2020-12-22T02:54:27Z" level=info msg="starting the server" addr=":3000"
time="2020-12-22T02:54:27Z" level=info msg="successfully pinged the remote server"
time="2020-12-22T02:54:27Z" level=info msg="polling the remote server" capacity=100 endpoint="http://172.31.223.247:10080" kind=pipeline type=kubernetes
~~~

​		至此平台已经搭建完成，后面就是使用了。

## 使用

> 我想了一下，使用drone的关键是配置`.drone.yml`，在这个文件里面会配置自动化流程的每个步骤。这个步骤里面有拉源码（git）、有打包（build）、有生成镜像（docker build），有部署（depoly kubernetes）。这样的话，那就得对整个流程的东西都有点了解，搭建排查整个流程的人应该对每个步骤有点了解或有点基础。

### 激活仓库

请转[启动Drone Server](###启动 Drone Server)

![image.png](https://i.loli.net/2021/01/14/bw4KS63rMBuR7ni.png)

除了上面的`Trusted`的问题之外，就是Configuration字段的配置。这个配置决定你的代码仓库用哪个文件来实现自动化流程。

### 代码仓库设置

~~~shell
#非自动化化
├── README.md
├── souce docde directory
│   ├── xxxx
│   └── src
│       └── reources code
└── log
    └── xxx
        ├── xxxx.log
        └── xxxx
            └── xxxx.log
~~~



~~~shell
#自动化
├── README.md
├── build
│   ├── Dockerfile
│   └── Project directory
│       └── xxxx
│           ├── bin
│           │   ├── xxxx
│           │   └── xxxxx.sh
│           ├── conf
│           │   └── ssss.conf
│           └──  start-sh
├── deploy
│   ├── .kube
│   │     └── config
│   ├── project-ds.yml
│   └── project_configmap.yml
├── souce docde directory
│   ├── xxxx
│   └── src
│       └── reources code
├── log
│   └── xxx
│       ├── xxxx.log
│       └── xxxx
│           └── xxxx.log
└── .drone.yml
~~~

对比前后，可以发现后者大概多了这几项目录或者文件： `build`目录、`deploy`目录、`.drone.yml`文件。

#### build

> 这个目录是用来打包容器镜像的目录，该目录下应该至少包含两部分内容：
>
> 1. Dockerfile： 打包镜像的文件
> 2. 项目启动的相关内容：

示例Dockerfile：

~~~shell
# This dockerfile demo for project build to docker images
# VERSION 2
# Command format: Instruction [arguments / command]

# 第一行必须指定基础容器，建议使用aipln类型的小容器
FROM tomcat:8

# LABEL (可选) 标签信息(自定义信息,多标签放一行)
LABEL app.maintainer=user_name
LABEL app.version="1.0" app.host='bestxiao.cn' description="这个app产品构建"

# ENV  (可选)环境变量(指定一个环境变量，会被后续 RUN 指令使用，并在容器运行时保持 
ENV JAVA_HOME /opt/java_jdk/bin
ENV PG_VERSION 9.3.4
ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH

# USER (可选) 指定运行容器时的用户名或 UID，后续的 RUN 也会使用指定用户,前面的RUN 不受影响
# RUN groupadd -r postgres && useradd -r -g postgres postgres 
USER postgres

# WORKDIT 后续的 RUN、CMD、ENTRYPOINT 指令配置容器内的工作目录
WORKDIR /path/to/workdir

# ADD/COPY 将外部文件copy到容器中。区别是ADD可以使用URL，还可以是tar
# COPY只能使用dockerfile所在目录
# ADD <src> <dest>
# COPY <src> <dest>
COPY target/tomcat-release.war /usr/local/tomcat/webapps/

# RUN 镜像的操作指令
# RUN <command> [“executable”, “param1”, “param2”]。
RUN echo “deb http://archive.ubuntu.com/ubuntu/ raring main universe” >> /etc/apt/sources.list
RUN apt-get update && apt-get install -y nginx
RUN mkdir /opt/deploy/
RUN echo “\ndaemon off;” >> /etc/nginx/nginx.conf

# EXPOSE 容器启动后需要暴露的端口
EXPOSE 22 80 8443 8080

# VOLUME 本地或其他容器挂载的挂载点，一般用来存放数据库和需要保持的数据等。
#VOLUME ["/data"]
VOLUME ["/data/postgres", "/other/path/"]


# ENTRYPOINT  容器启动后执行命令，不会被docker run提供的参数覆盖，只能有一个ENTRYPOINT,
# 多个ENTRYPOINT，以最后一个为准
#ENTRYPOINT [“executable”, “param1”, “param2”]
#ENTRYPOINT command param param2
ENTRYPOINT echo "helloDocker"  

# 容器启动时执行指令,每个 Dockerfile 只能有一条 CMD 命令
#CMD [“executable”, “param1”, “param2”] 使用 exec 执行，推荐方式。
#CMD command param1 param2 在 /bin/sh 中执行，提供给需要交互的应用。
#CMD [“param1”, “param2”] 提供给 ENTRYPOINT 的默认参数。
CMD /usr/sbin/nginx


# ONBUILD 配置当所创建的镜像作为其他新创建镜像的基础镜像时，所执行的操作指令。例如，Dockerfile 使用如下的内容创建了镜像 image-A。-- 很少使用

# ONBUILD ADD . /app/src
# ONBUILD RUN /usr/local/bin/python-build –dir /app/src
~~~

选自：[Docker：DockerFile详解与实例](https://www.cnblogs.com/nhdlb/p/12516409.html)

#### deploy目录

> 该目录是用来部署服务用的，该目录应包含两项内容：
>
> 1. 部署编排用的yaml文件；
> 2. K8S认证的config文件：需要使用认证文件完成应用的部署更新。

示例yaml：

~~~shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drone
  namespace: drone
  labels:
    app.kubernetes.io/name: drone
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: drone
  template:
    metadata:
      labels:
        app.kubernetes.io/name: drone
    spec:
      containers:
      - name: runner
        image: drone/drone-runner-kube:linux-amd64
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
        env:
        - name: DRONE_RPC_HOST
          value: 172.31.223.247:10080
        - name: DRONE_RPC_PROTO
          value: http
        - name: DRONE_RPC_SECRET
          value: e3571xxxxx59decf00a5baeb0486
        - name: DRONE_NAMESPACE_DEFAULT
          value: "drone"
      nodeSelector:
        drone-runner: "true"
      restartPolicy: Always
~~~

#### .drone.yml

​		这个文件是大头，这个文件名，须和drone server 项目配置的`Configuration`配置项相同，否则也触发不了自动化流程。这个drone.yml属于drone很关键的一步，这里好好的讲一下。

​		这个文件放的位置也必须是代码仓库的根位置。

示例：

~~~shell
kind: pipeline
type: kubernetes #pipeline的种类，不同的pipeline的种类对应不同的runner
name: serviceName

metadata:
#这个指定runner起的pipeline的pod在哪个命名空间，
#默认是default，这个默认的配置可以在起runner的时候通过环境变量“DRONE_NAMESPACE_DEFAULT”指定
#注意，这有个坑，runner在命名空间下创建pod需要授权的，要是对应的ns下没有权限的话，是创建不了pipeline的pod的
  namespace:xxxx 

# base 为该 pipeline 各个 step 共享 volume（本身默认为 /drone，这里显示指定）
# path 是一个基于 base 的相对路径，存储 clone 的源码（本身默认为 src，这里显示指定）
# 所以 base/path 即为该 pipeline 用于存储 clone 代码的地方
# 以上为文档所属，但是与具体不符，path 需要使用绝对路径。
workspace:
  path: /drone/src

trigger:
  # 当下述全部条件同时成立的时候才执行该 step
  # 注意，branch 条件不能和 tag event 一起用，因为 tag 事件本身不包含任何分支信息
  # 如果要控制每一步的执行条件，可以在对应 step 下面增加 when，剩下配置与 trigger 类似
  event:
    - tag

# 禁止默认的 clone 操作
clone:
  disable: true

steps:
  - name: clone #步骤名字
    image: dockerhub.azk8s.cn/alpine/git:1.0.7 #用哪个镜像完成任务
    pull: if-not-exists #镜像下载策略
    commands: #自定义的名字
      - echo "10.103.57.10 git.xxx.com" > /etc/hosts
      - git clone https://git.xxxx.com/CBG_AIM/xxxx_server.git .
      - git checkout $DRONE_COMMIT
  - name: maven-build
    image: dockerhub.azk8s.cn/library/maven:3.6.3-openjdk-11
    pull: if-not-exists
    # 挂在 volumes 前提是，该 pipeline 处理的 repo 必须是 trusted 模式，这个需要
    # 在启动 drone-server 时候将 repo 的某个 user 设置为 admin，然后在 drone-server
    # web 页面打开该 repo 的 setting 进行设置。
    # 只有设置为 trusted 才能挂载 volumes，或者设置 privileged 为 true（即允许以 root 权限访问 host）
    volumes:
      # 将 maven repo 映射到 host，这样此后再只需 maven 时一些下载过的依赖无须再次下载，以加速 build
      - name: mvn-repo
        path: /root/.m2/repository
      - name: product-repo
        path: /tmp
    commands:
      # 下面全部命令会打包成脚本作为容器启动的 Entrypoint
      - env
      - mvn clean install -Dmaven.test.skip=true
      - pwd
      - ls
      - cp target/xxx_server-0.0.1-SNAPSHOT.jar build/xxxx_server/
      - mkdir -p /tmp/$DRONE_REPO_NAMESPACE/$DRONE_REPO_NAME/${DRONE_TAG:-$(echo $DRONE_BUILD_NUMBER)}
      - cp target/*.jar /tmp/$DRONE_REPO_NAMESPACE/$DRONE_REPO_NAME/${DRONE_TAG:-$(echo $DRONE_BUILD_NUMBER)}/
  - name: docker-build
    image: dockerhub.azk8s.cn/plugins/docker:18.09
    pull: if-not-exists
    # 必须是 root 权限，否则报错，如 Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
    privileged: true
    settings:
      insecure: true
      registry: xxxharbor.voiceads.cn
      username: user
      password: secretxxxx
      context: /drone/src/build
      dockerfile: /drone/src/build/Dockerfile
      repo: xxxharbor.voiceads.cn/xxxx_server/test
      auto_tag: false
      tags:
        - ${DRONE_TAG}
        - latest
  - name: deploy
    image: dockerhub.azk8s.cn/bitnami/kubectl:1.16.3
    pull: if-not-exists
    privileged: true
    commands:
      - env
      - export KUBECTL_CONF=/drone/src/deploy/.kube/config
      - export DEPLOY_DIR=/drone/src/deploy
      #      - sed -i -e "s/latest/${DRONE_TAG:-$(echo $DRONE_BUILD_NUMBER)}/g" $DEPLOY_DIR/adx-inner-server.yml
      - kubectl --kubeconfig=$KUBECTL_CONF apply -f $DEPLOY_DIR/xxxx-server.yml --record
      #      - kubectl --kubeconfig='/tmp/.kube/config' wait --timeout=100s --for=condition=complete -f adx-inner-server.yml
      - kubectl --kubeconfig=$KUBECTL_CONF -n xxxx rollout status ds xxx-server --timeout=300s -w
  - name: notify
    image: 172.16.xx.xxx:5000/plugins/drone-email:0.3.0
    pull: if-not-exists
    settings:
      host: mail.xxx.com
      port: 465
      username: user
      password: passwd
      from: user@xxx.com
      recipients:
        - user1@xxx.com
        - user2@xxx.com
        - user3@xxx.com
      recipients_only: true
      skip_verify: true
    when:
      status:
        - success
        - failure
volumes:
  - name: mvn-repo
    host:
      path: /data/maven/repository
  - name: product-repo
    host:
      path: /data/product/
~~~





# 