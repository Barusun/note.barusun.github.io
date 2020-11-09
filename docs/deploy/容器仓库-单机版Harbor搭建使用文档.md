# 容器仓库-单机版Harbor搭建使用文档

> 部署机器：172.21.xx.xx
>
> 部署目录：/data/harbor
>
> 系统信息：Ubuntu 16.04.6 LTS \n \l
>
> 内核信息：Linux 4.4.0-116-generic
>
> 内存信息：
>
> ~~~shell
> # free -h
>               total        used        free      shared  buff/cache   available
> Mem:            62G         23G        8.8G        695M         30G         36G
> Swap:            0B          0B          0B
> ~~~
> 
>存储信息：
> 
>~~~shell
> # df -Th |grep -v /var/lib
> Filesystem     Type      Size  Used Avail Use% Mounted on
> /dev/sda2      ext4      439G  121G  296G  29% /
> /dev/sdb1      ext4      440G  108G  310G  26% /data
> /dev/sda1      ext4      922M   57M  802M   7% /boot
> ~~~
> 
> 容器信息：
> 
> ~~~shell
> # docker version
> Client:
>Version:           18.06.0-ce
> API version:       1.38
>Go version:        go1.10.3
> Git commit:        0ffa825
> Built:             Wed Jul 18 19:11:02 2018
> OS/Arch:           linux/amd64
>  Experimental:      false
>  
>  Server:
>  Engine:
>  Version:          18.06.0-ce
>  API version:      1.38 (minimum version 1.12)
>  Go version:       go1.10.3
> Git commit:       0ffa825
> Built:            Wed Jul 18 19:09:05 2018
>  OS/Arch:          linux/amd64
>   Experimental:     false
>   ~~~

## 安装docker-compose

~~~shell
curl -L https://github.com/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
~~~

~~~shell
chmod +x /usr/local/bin/docker-compose
~~~

~~~shell
# docker-compose version
docker-compose version 1.25.5, build 8a1c60f6
docker-py version: 4.1.0
CPython version: 3.7.5
OpenSSL version: OpenSSL 1.1.0l  10 Sep 2019
~~~



## 安装harbor

### 下载资源

~~~shell
mkdir /data/harbor -p
cd /data/harbor/
wget https://github.com/vmware/harbor/releases/download/v1.10.2/harbor-online-installer-v1.10.2.tgz
~~~

> 版本：1.10.2可以根据部署时候的版本

### 解压缩

~~~shell
cd /data/harbor/
tar -zxvf harbor-online-installer-v1.10.2.tgz
~~~



### 准备证书

~~~shell
mkdir /data/harbor/ca/
将域名的证书传到该节点的该目录下
~~~

### 修改配置文件

~~~shell
vim /data/harbor/harbor.yml
1.hostname: harbor.xxx.cn
2.port: 1443 #因为该节点443端口被占用，所以用1443
3.certificate: /data/harbor/ca/xxx.crt
4.private_key: /data/harbor/ca/xxx.key
5.location: /data/harbor/log
~~~



### 开始安装

~~~shell
# ./install.sh 

[Step 0]: checking if docker is installed ...

Note: docker version: 18.06.0

[Step 1]: checking docker-compose is installed ...

Note: docker-compose version: 1.25.5


[Step 2]: preparing environment ...

[Step 3]: preparing harbor configs ...
prepare base dir is set to /data/harbor
Clearing the configuration file: /config/jobservice/config.yml
Clearing the configuration file: /config/jobservice/env
Clearing the configuration file: /config/nginx/nginx.conf
Clearing the configuration file: /config/db/env
Clearing the configuration file: /config/core/app.conf
Clearing the configuration file: /config/core/env
Clearing the configuration file: /config/log/rsyslog_docker.conf
Clearing the configuration file: /config/log/logrotate.conf
Clearing the configuration file: /config/registry/config.yml
Clearing the configuration file: /config/registry/root.crt
Clearing the configuration file: /config/registryctl/config.yml
Clearing the configuration file: /config/registryctl/env
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
loaded secret from file: /secret/keys/secretkey
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir


Note: stopping existing Harbor instance ...
Stopping harbor-jobservice ... done
Stopping harbor-core       ... done
Stopping harbor-db         ... done
Stopping registryctl       ... done
Stopping harbor-portal     ... done
Stopping registry          ... done
Stopping redis             ... done
Stopping harbor-log        ... done
Removing harbor-jobservice ... done
Removing nginx             ... done
Removing harbor-core       ... done
Removing harbor-db         ... done
Removing registryctl       ... done
Removing harbor-portal     ... done
Removing registry          ... done
Removing redis             ... done
Removing harbor-log        ... done
Removing network harbor_harbor


[Step 4]: starting Harbor ...
Creating network "harbor_harbor" with the default driver
Creating harbor-log ... done
Creating harbor-portal ... done
Creating redis         ... done
Creating harbor-db     ... done
Creating registryctl   ... done
Creating registry      ... done
Creating harbor-core   ... done
Creating nginx             ... done
Creating harbor-jobservice ... done
✔ ----Harbor has been installed and started successfully.----
~~~

> 这里harbor使用的镜像都在之前下载好了，要是没有镜像的话，会有个下载镜像的过程，可能会非常慢。要是中间出现错误，可直接重新执行：./install.sh

### 检查

~~~shell
# docker-compose ps -a
      Name                     Command                  State                          Ports                    
----------------------------------------------------------------------------------------------------------------
harbor-core         /harbor/harbor_core              Up (healthy)                                               
harbor-db           /docker-entrypoint.sh            Up (healthy)   5432/tcp                                    
harbor-jobservice   /harbor/harbor_jobservice  ...   Up (healthy)                                               
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp                   
harbor-portal       nginx -g daemon off;             Up (healthy)   8080/tcp                                    
nginx               nginx -g daemon off;             Up (healthy)   0.0.0.0:80->8080/tcp, 0.0.0.0:1443->8443/tcp
redis               redis-server /etc/redis.conf     Up (healthy)   6379/tcp                                    
registry            /home/harbor/entrypoint.sh       Up (healthy)   5000/tcp                                    
registryctl         /home/harbor/start.sh            Up (healthy)                                               
root@bjb-xxx-052061:/data/harbor# docker-compose ps -a
      Name                     Command                  State                          Ports                    
----------------------------------------------------------------------------------------------------------------
harbor-core         /harbor/harbor_core              Up (healthy)                                               
harbor-db           /docker-entrypoint.sh            Up (healthy)   5432/tcp                                    
harbor-jobservice   /harbor/harbor_jobservice  ...   Up (healthy)                                               
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp                   
harbor-portal       nginx -g daemon off;             Up (healthy)   8080/tcp                                    
nginx               nginx -g daemon off;             Up (healthy)   0.0.0.0:80->8080/tcp, 0.0.0.0:1443->8443/tcp
redis               redis-server /etc/redis.conf     Up (healthy)   6379/tcp                                    
registry            /home/harbor/entrypoint.sh       Up (healthy)   5000/tcp                                    
registryctl         /home/harbor/start.sh            Up (healthy)          
~~~

> 有web页面的话可以使用web界面登陆验证一下。使用的登陆地址为：`https://harbor.voiceads.cn:1443/`，用户名密码默认为：`admin/Harbor12345`。可以登陆之后修改。



## 使用

1. 登陆

> 为了方便在办公网访问，我在59.106上做了nginx代理，代理配置如下：
>
> ~~~shell
> server {
>     listen 443;
>     server_name harbor.xxx.cn;
>     error_page 500 502 503 504 /50x.html;
>     ssl on;
>     ssl_certificate /usr/local/nginx/ssl/ca/xxx.crt;
>     ssl_certificate_key /usr/local/nginx/ssl/ca/xxx.key;
>     location = /favicon.ico {
>         return 404;
>     }
>     location = /50x.html {
>         root /var/www;
>     }
>     location ~ \.*$ {
>         proxy_set_header Host $host;
>         proxy_set_header X-Real-IP $remote_addr;
>         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
>         proxy_pass https://172.21.xx.xx:1443;
>     }
> }
> ~~~
>
> hosts添加解析
>
> 172.16.59.106 harbo.xxx.cn

![image.png](https://i.loli.net/2020/11/09/z6jBiMLKdpDlu9P.png)

2. 新建项目

![image.png](https://i.loli.net/2020/11/09/Ruz8d4gaV5oE3IO.png)

![image.png](https://i.loli.net/2020/11/09/8aqVt46diNJRCFY.png)



3. 登陆仓库

   ~~~shell
   docker login -u admin -p ${registry_passwd} harbor.xxx.cn/xxx-registry
   ~~~

4. 打镜像

   ~~~shell
   docker build -f Dockerfile.xxx -t harbor.xxx.cn:1443/xxx-registry/xxx:xxx_Build1634 .
   ~~~

5. 推镜像

   ~~~shell
   docker push 
   ~~~

6. 创建K8S使用密钥

   ~~~shell
   kubectl create secret docker-registry xxx-harbor-secret --namespace=kube-xxx \
   --docker-server=harbor.xxx.cn:1443/xxx-registry --docker-username=admin \
   --docker-password='xxx' --docker-email=xxx@xxk.com
   ~~~

7. 查看

   ~~~shell
   # kubectl -n kube-xxx get secret
   NAME                         TYPE                                  DATA      AGE
   xxx-harbor-secret            kubernetes.io/dockerconfigjson        1         18h
   ~~~

8. 修改yaml

   ~~~shell
   spec.template.sepc 添加：
         imagePullSecrets:
         - name: xxx-harbor-secret
   ~~~

> 即可免密使用了



## 踩坑记

搭的这个版本，docker-compose默认的网卡地址是：172.22.0.0/16段的。部署完成之后，会创建一个harbor的网卡：可使用：ifconfig 和docker network ls 查看，如果内网有冲突的网段，会导致这个网段到harbor不通

