# Linux(Centos&Ubuntu)安装指定版本MYSQL

[TOC]



## Ubuntu16.04指定安装Mysql版本---5.7.22

​										

### 创建工作目录

~~~shell
mekdir /home/xiaoxiao/mysql
~~~

### 下载安装包

安装MySQL5.7.20（或任意版本）时，需要去MySQL官网下载deb包

官网地址：[官网链接](http://repo.mysql.com/apt/ubuntu/pool/mysql-5.7/m/mysql-community/ )

> 需要在如上链接中，查找需要安装的Mysql版本

#### 下载需要的mysql安装包

~~~shell
cd /home/xiaoxiao/mysql
wget http://repo.mysql.com/apt/ubuntu/pool/mysql-5.7/m/mysql-community/mysql-client_5.7.22-1ubuntu16.04_amd64.deb
wget http://repo.mysql.com/apt/ubuntu/pool/mysql-5.7/m/mysql-community/mysql-common_5.7.22-1ubuntu16.04_amd64.deb
wget http://repo.mysql.com/apt/ubuntu/pool/mysql-5.7/m/mysql-community/mysql-community-client_5.7.22-1ubuntu16.04_amd64.deb
wget http://repo.mysql.com/apt/ubuntu/pool/mysql-5.7/m/mysql-community/mysql-community-server_5.7.22-1ubuntu16.04_amd64.deb
~~~

#### 下载依赖包

~~~
wget https://mirrors.aliyun.com/ubuntu/pool/main/liba/libaio/libaio1_0.3.109-4_amd64.deb
wget http://archive.ubuntu.com/ubuntu/pool/universe/m/mecab/libmecab2_0.996-1.2ubuntu1_amd64.deb
~~~

### 安装数据库

> note: 需要按此顺序安装，因为彼此包之间有依赖

~~~shell
dpkg -i mysql-common_5.7.22-1ubuntu16.04_amd64.deb 
dpkg -i libaio1_0.3.109-4_amd64.deb
#mysql-community-client依赖上面libaio这个包
dpkg -i mysql-community-client_5.7.22-1ubuntu16.04_amd64.deb 
dpkg -i mysql-client_5.7.22-1ubuntu16.04_amd64.deb 
dpkg -i libmecab2_0.996-1.2ubuntu1_amd64.deb
dpkg -i mysql-community-server_5.7.22-1ubuntu16.04_amd64.deb
（安装最后一个包的时候需要输入root密码）
~~~

> #mysql-community-server_5.7.22依赖上面libmecab，且对版本有要求，我当时装的阿里云0.996-1.1的版本
> #但是提示：mysql-community-server depends on libmecab2 (>= 0.996-1.2ubuntu1)

### 配置数据库

#### 解除绑定IP：127.0.0.1

修改文件：mysqld.cnf

~~~
vim /etc/mysql/mysql.conf.d/mysqld.cnf
 将：bind-address    = 127.0.0.1
改为：bind-address    = 0.0.0.0
~~~

#### 重启mysql服务

~~~shell
systemctl restart mysql.service
#检查监听端口
ss -ntlp |grep mysql
#显示如下
LISTEN     0      80           *:3306                     *:*                   users:(("mysqld",pid=40258,fd=29))
~~~

### 授权root远程登陆

~~~shell
##连接数据库
mysql -uroot –p
##授权
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
##刷新权限
FLUSH PRIVILEGES;

~~~

## Centos 7.2安装指定版本Mysql版本--5.7.21

### 创建工作目录

~~~shell
mekdir /home/xiaoxiao/mysql
~~~

### 下载安装包

安装MySQL5.7.21（或任意版本）时，需要去MySQL官网下载RPM包

官网地址：[官网链接](<http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/>)；

> 需要在如上链接中，查找需要安装的Mysql版本

~~~shell
cd /home/xiaoxiao/mysql
wget http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-libs-5.7.21-1.el7.x86_64.rpm
wget http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-common-5.7.21-1.el7.x86_64.rpm
wget http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-server-5.7.21-1.el7.x86_64.rpm
wget http://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-client-5.7.21-1.el7.x86_64.rpm
~~~

### 检查Mariadb

> **NOTE:** Centos 7 已经不支持mysql，所以内部集成了mariadb，而安装mysql的话会和mariadb的文件冲突，所以需要先卸载掉mariadb，以下为卸载mariadb的步骤

~~~shell
rpm -qa | grep mariadb
~~~

如果有，请使用如下命令卸载mariadb

~~~shell
#例：
rpm -e mariadb-libs-5.5.44-2.e17.centos.x86_64 -nodeps
~~~

### 安装依赖包

~~~shell
yum -y install libaio-devel.x86_64
~~~

### 依次安装RPM包

> **NOTE：**需要按照如下次序安装rpm包。因为彼此之间有依赖

~~~shell
rpm -ivh mysql-community-common-5.7.21-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.21-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.21-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.21-1.el7.x86_64.rpm
~~~

安装过程如下图

![mysql安装过程图.png](https://i.loli.net/2019/09/03/kauhwObtV17nj8E.png)

### 启动Mysql

~~~shell
systemctl enable mysqld
systemctl start mysqld
systemctl status mysqld
~~~

### 登陆Mysql 并初始化密码

#### 查看初始化密码

> 初始化的密码在：`/var/log/mysqld.log`里。

~~~shell
cat /var/log/mysqld.log |grep "root@localhost"
~~~

#### 修改密码：

~~~shell
#登陆mysql
mysql -u root -p
#输入以上得到的密码
mysql>set password=password("xxxxxx");
#提示：
Quey OK, 0 rows affected, 1 warning(0.00 sec)
~~~

### ERROR

#### 1. 与mariadb冲突

如下图所示：

![mariadb冲突error.png](https://i.loli.net/2019/09/03/sgqZGSncxRClW3w.png)

##### 解决：

如上`检查Mariadb`步骤，使用`rpm -e`卸载相关包。

#### 依赖出错

​	安装的过程中提示有依赖。

##### 解决

本次安装没有遇到，遇到后可根据提示安装相关依赖后，重新安装即可。

