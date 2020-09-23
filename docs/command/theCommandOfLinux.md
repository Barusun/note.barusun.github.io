​	

# Linux 命令

## Table of Contents

- [A](#A)
- [B](#B)
- [C](#C)
- [D](#D)
- [E](#E)
- [F](#F)
- [G](#G)
- [H](#H)
- [I](#I)
- [J](#J)
- [K](#K)
- [L](#L)
- [M](#M)
- [N](#N)
- [O](#O)
- [P](#P)
- [Q](#Q)
- [R](#R)
- [S](#S)
- [T](#T)
- [U](#U)
- [V](#V)
- [W](#W)
- [X](#X)
- [Y](#Y)
- [Z](#Z)

- [Kubernetes](#Kubernetes)
- [组合命令](#组合命令)
- [脚本](#脚本)

## A
- [Table of Contents](#Table-of-Contents)

### anssible

#### 安装

~~~shell
apt -y install sshpass ansible
~~~

~~~shell
#修改hosts文件，添加主机
[master]
192.168.137.167 ansible_ssh_user="root" ansible_ssh_pass="x"

# 修改ansible 配置文件，
# vim /etc/ansible/ansible.cfg 
host_key_checking = False
#不然报以下error
# Please add this host's fingerprint to your known_hosts file to manage tis host

~~~

#### 剧本

~~~shell
---
- hosts: Gf_hosts
  remote_user: root

  tasks:
    - name: Push_Jdk8
      copy: src=jdk1.8.0_131.tar.gz dest=/home/ops/
      when: handle == "init" or handle == "setJdk"

    - name: Push_Jdk11
      copy: src=openjdk-11.0.2_linux-x64_bin.tar.gz dest=/home/ops/
      when: handle == "init" or handle == "setJdk"

    - name: Push_Agent
      copy: src=agent.tar.gz dest=/home/ops
      when: handle == "init" or handle == "setAgent"

    - name: Push_gf
      copy: src=gf.sh dest=/home/ops/gf.sh mode=0775
      when: ansible_machine == "x86_64" and ansible_os_family =="Debian" and ( ansible_distribution_version == "16.04" or ansible_distribution_version == "18.04" )
      register: result

    - name: Run_gf
      shell: /bin/bash /home/ops/gf.sh {{ handle }} {{ type }}
      when: not ( result is skip or result is failure )
      register: check
    - name: Show_gf
      debug: var=check.stdout verbosity=0

    - name: Rm_gf
      shell: rm /home/ops/gf.sh

    - name: Restart_sshd
      shell: systemctl restart sshd
      when: handle == "init" or handle == "setProfile"

    - debug:
        msg: "目标系统版本非Ubuntu16/Ubuntu18"
      when: result is ski

~~~

#### 命令

~~~shell
#执行本地脚本
ansible dsp-node -i hosts/dsp-monitor-hosts -m script  -a 'netstat.sh'
~~~

参考文档：[ansible之shell和script模块](http://www.mamicode.com/info-detail-1312187.html)

### arp 表查看

~~~shell
arp -a 
~~~



### apt 卸载软件

~~~shell
#ubuntu
查看已安装
apt list --installed
或者
dpkg -l|grep 'ftp'
卸载
apt remove vsftpd
或者
apt-get remove vsftpd
添加--purge选项，指的是卸载是把程序包的配置文件也删掉。
apt remove --purge vsftpd
或者
apt-get remove --purge vsftpd

~~~

### apt锁定软件

~~~shell
#锁定docker
apt-mark hold docker-ce
docker-ce set on hold.
#查看锁定
# apt-mark showhold
docker-ce
~~~



### AWK命令

~~~shell
awk [options] 'program' file
[options]:
	-F: 输入时用到的分割符，默认空格
	-v: 指定变量 var=value
program：
	PATERN[action STATEMENTS]
	action:
		print:
			print item1,item2
			1、逗号分隔符；
			2、$符在引号里做字符串输出；可以字符串，可以字符（对输出格式做调整）

#变量
	1.内建变量：
		FS: 输入分隔符，默认为空格；
		OPS：输出分割符，默认是空格； awk -v OPS=':' /etc/passwd
		RS：输入时行分隔符，默认为换行符；
		ORS：输出时的行分割符，默认为换行符
		NF：每行的字段数量；awk ‘{print NF}’ /etc/fstab(不要加$)
						awk ‘{print $NF}’ /etc/fstab	(打印每行最后一个字段，NF是一个数值)
	    NR： 文件的每行的行数
	    	awk '{print NR,$0}' /etc/hosts 在每行前面加行号
	    	
		ARGC： 命令行参数个数 awk '{print ARGC' /etc/hosts #输出2（awk，/etc/hosts）
		ARGV: 数组，保存的时命令行所给定的各参数 awk '{print ARGV[0]}' /etc/hosts
		
	2、自定义变量
		（1）-v val=value (区分大小写)
			awk -v test="hello awk" 'BEGIN{print test}' (BEGIN输出一次)
		（2）在program中直接定义
        	awk 'BEGIN{test="hello awk";print test}'
     3、printf命令（格式化输出）
     	格式化输出：printf "FORMAT"，item1，item2...
     		（1）、FORMAT必须给出；
     		（2）、不会自动换行，，需要显示的给出换行控制符，\n
     		 (3) FORMAT中需要分别为后面的每个item指定一个格式化符号；
     		格式符：
					%c: 显示字符的ASCII码；
					%d, %i: 显示十进制整数；
					%e, %E: 科学计数法数值显示；
					%f：显示为浮点数；
					%g, %G：以科学计数法或浮点形式显示数值；
					%s：显示字符串；
					%u：无符号整数；
					%%: 显示%自身；		
			实例：awk -F: '{printf "Username:%s UUID:%s\n",$1,$3}' /etc/passwd
			
			修饰符：
				#[.#] 第一个#表示字符的宽度，点后的#表示小数后的精度；如：%3.1f
				-: 左对齐；（默认右对齐）
				+： 显示数值的符号
				
		4、操作符
			算数操作符： x+y x-y x%y x*y  x^y x/y
			字符串操作符： 没有符号的操作符
			比较操作符：
				>, >=, <, <=, !=, ==

			模式匹配符：
				~：是否匹配
				!~：是否不匹配

			逻辑操作符：
				&&
				||
				!

			函数调用：
				function_name(argu1, argu2, ...)

			条件表达式：
				selector?if-true-expression:if-false-expression

				# awk -F: '{$3>=1000?usertype="Common User":usertype="Sysadmin or SysUser";printf "%15s:%-s\n",$1,usertype}' /etc/passwd

			
			awk -F: '{$3>=1000?userType="Common user":userType="System user";printf "%-17s %15s\n",$1,userType}' /etc/passwd
			（判断passwd文件里第三个UUID是否大于1000，大于1000则userType赋值commonuser）
     	5、PATTERN
     		（1）empty： 匹配每一行
     		（2）/regular expression/：仅处理能够被此模式匹配到的行，支持正则表达式
     			awk '/^UUID/{print $1}' /etc/fstab #以UUID开头的行
     			awk '!/^UUID/{print $1}' /etc/fstab #不以UUID开头的行
     			awk '/UUID/{print $0}' /etc/fstab  #包含UUID的行
     			awk '!/UUID/{print $0}' /etc/fstab   #不包含UUID的行
     		（3）relational expression: 关系表达式：结果有真有假，为真的才会被处理
     			真：结果为非0值；非空字符串为真；
     			awk -F: '$3>=1000{print $1,$3}' /etc/passwd
     			awk -F: '$NF=="/bin/bash"{print $1,$NF}' /etc/passwd
     			awk -F: '$NF~/bash$/{print $1,$NF}' /etc/passwd #模式匹配：最后一段的值，匹配以bash结尾
     			awk -F: 'NR>=17&&NR<=27{print $1}' /etc/passwd
     		（4）line range 指定行范围
     			/startExpression/，/endExpression/ 不支持：2，5模式
     			awk '{print NR,$0}' /etc/passwd | awk -F: '/^17/,/^27/{print $1}'
     			awk -F: 'NR>=17&&NR<=27{print $1}' /etc/passwd
     		（5）BEGIN/END模式
     			BEGIN{}：仅在开始处理文件中的文本之前执行一次（打印表头）
     			END{}: 仅在文本处理完成后执行一次
		6、常用的action

			(1) Expressions
			(2) Control statements：if, while等；
			(3) Compound statements：组合语句；
			(4) input statements
			(5) output statements

		7、控制语句

			if(condition) {statments} 
			if(condition) {statments} else {statements}
			while(conditon) {statments}
			do {statements} while(condition)
			for(expr1;expr2;expr3) {statements}
			break
			continue
			delete array[index]
			delete array
			exit 
			{ statements }

			7.1 if-else

				语法：if(condition) statement [else statement]
				~]# awk -F: '{if($3>=1000) {printf "Common user: %s\n",$1} else {printf "root or Sysuser: %s\n",$1}}' /etc/passwd
				#多个语句块用{}括起来
				~]# awk -F: '{if($NF=="/bin/bash") print $1}' /etc/passwd

				~]# awk '{if(NF>5) print $0}' /etc/fstab

				~]# df -h | awk -F[%] '/^\/dev/{print $1}' | awk '{if($NF>=20) print $1}'

				使用场景：对awk取得的整行或某个字段做条件判断；

			7.2 while循环
				语法：while(condition) statement
					条件“真”，进入循环；条件“假”，退出循环；

				使用场景：对一行内的多个字段逐一类似处理时使用；对数组中的各元素逐一处理时使用；

				~]# awk '/^[[:space:]]*linux16/{i=1;while(i<=NF) {print $i,length($i); i++}}' /etc/grub2.cfg

				~]# awk '/^[[:space:]]*linux16/{i=1;while(i<=NF) {if(length($i)>=7) {print $i,length($i)}; i++}}' /etc/grub2.cfg

			7.3 do-while循环
				语法：do statement while(condition)
					意义：至少执行一次循环体

			7.4 for循环
				语法：for(expr1;expr2;expr3) statement

					for(variable assignment;condition;iteration process) {for-body}

				~]# awk '/^[[:space:]]*linux16/{for(i=1;i<=NF;i++) {print $i,length($i)}}' /etc/grub2.cfg

				特殊用法：
					能够遍历数组中的元素；
						语法：for(var in array) {for-body}

			7.5 switch语句
				语法：switch(expression) {case VALUE1 or /REGEXP/: statement; case VALUE2 or /REGEXP2/: statement; ...; default: statement}

			7.6 break和continue
				break [n]
				continue

			7.7 next 控制awk自身对每行的循环

				提前结束对本行的处理而直接进入下一行；

				~]# awk -F: '{if($3%2!=0) next; print $1,$3}' /etc/passwd

		8、array

			关联数组：array[index-expression]

				index-expression:
					(1) 可使用任意字符串；字符串要使用双引号；
					(2) 如果某数组元素事先不存在，在引用时，awk会自动创建此元素，并将其值初始化为“空串”；

					若要判断数组中是否存在某元素，要使用"index in array"格式进行；

					weekdays[mon]="Monday"

				若要遍历数组中的每个元素，要使用for循环；
					for(var in array) {for-body}

					~]# awk 'BEGIN{weekdays["mon"]="Monday";weekdays["tue"]="Tuesday";for(i in weekdays) {print weekdays[i]}}'

					注意：var会遍历array的每个索引；
					state["LISTEN"]++
					state["ESTABLISHED"]++

					~]# netstat -tan | awk '/^tcp\>/{state[$NF]++}END{for(i in state) { print i,state[i]}}'

					~]# awk '{ip[$1]++}END{for(i in ip) {print i,ip[i]}}' /var/log/httpd/access_log

					练习1：统计/etc/fstab文件中每个文件系统类型出现的次数；
					~]# awk '/^UUID/{fs[$3]++}END{for(i in fs) {print i,fs[i]}}' /etc/fstab

					练习2：统计指定文件中每个单词出现的次数；
					~]# awk '{for(i=1;i<=NF;i++){count[$i]++}}END{for(i in count) {print i,count[i]}}' /etc/fstab

		9、函数

			9.1 内置函数
				数值处理：
					rand()：返回0和1之间一个随机数；

				字符串处理：
					length([s])：返回指定字符串的长度；
					sub(r,s,[t])：以r表示的模式来查找t所表示的字符中的匹配的内容，并将其第一次出现替换为s所表示的内容；
					gsub(r,s,[t])：以r表示的模式来查找t所表示的字符中的匹配的内容，并将其所有出现均替换为s所表示的内容；

					split(s,a[,r])：以r为分隔符切割字符s，并将切割后的结果保存至a所表示的数组中；

					~]# netstat -tan | awk '/^tcp\>/{split($5,ip,":");count[ip[1]]++}END{for (i in count) {print i,count[i]}}'

			9.2 自定义函数

				《sed和awk     		
~~~

文件内容转置：

~~~shell
# cat file.txt
name age
alice 21
ryan 30
# awk '{for(i=1;i<=NF;i++)a[NR,i]=$i}END{for(j=1;j<=NF;j++)for(k=1;k<=NR;k++)printf k==NR?a[k,j] RS:a[k,j] FS}' file.txt 
name alice ryan
age 21 30
~~~

~~~shell
#统计每行字符数小于26的行有多少个
awk '{if(length($0)<26){count++;}} END {print "行数: ",count}' tmp.txt
~~~

## B
- [Table of Contents](#Table-of-Contents)

### blkid（查看设备UUID）

~~~shell
blkid /dev/sdb1
~~~



## C
- [Table of Contents](#Table-of-Contents)

### curl命令

~~~shell
curl -o /dev/null -w %{time_namelookup}/%{time_connect}/%{time_starttransfer}/%{time_total}/%{speed_download}"\n" "http://www.taobao.com"

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 28774    0 28774    0     0  1145k      0 --:--:-- --:--:-- --:--:-- 7550k
0.014::0.016::0.020::0.025::1173060.000
-o：把curl 返回的html、js 写到垃圾回收站[ /dev/null]

-s：去掉所有状态
-w：按照后面的格式输出

time_namelookup：DNS 解析域名[www.taobao.com]的时间

time_commect：client和server端建立TCP 连接的时间

time_starttransfer：从client发出请求；到web的server 响应第一个字节的时间

time_total：client发出请求；到web的server发送会所有的相应数据的时间

speed_download：下周速度 单位 byte/s
0.014: DNS 服务器解析www.taobao.com 的时间单位是s
0.015: client发出请求，到c/s 建立TCP 的时间；里面包括DNS解析的时间
0.018: client发出请求；到s响应发出第一个字节开始的时间；包括前面的2个时间
0.019: client发出请求；到s把响应的数据全部发送给client；并关闭connect的时间
~~~

### cat 

~~~shell
#文件内容查看
cat file1 从第一个字节开始正向查看文件的内容 
tac file1 从最后一行开始反向查看一个文件的内容 
cat -n file1 标示文件的行数 
more file1 查看一个长文件的内容 

head -n 2 file1 查看一个文件的前两行 
tail -n 2 file1 查看一个文件的最后两行 
tail -n +1000 file1  从1000行开始显示，显示1000行以后的
cat filename | head -n 3000 | tail -n +1000  显示1000行到3000行
cat filename | tail -n +3000 | head -n 1000  从第3000行开始，显示1000(即显示3000~3999行)
~~~



## D
- [Table of Contents](#Table-of-Contents)

### Docker

~~~shell
#添加阿里源
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 7EA0A9C3F273FCD8
#查看可安装的版本
apt-cache madison docker-ce
#安装指定版本镜像
apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu
#下载镜像
docker pull prom/prometheus:v2.10.0
#查看镜像
docker images -a
#修改docker工作路径
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --graph /data/docker
#删除docker ps -a 为exit状态的容器
docker rm -v $(docker ps -aq -f status=exited)
#清理docker空间（没有使用的镜像都会被清理）
docker system prune -af
#查看docker空间
docker system df
# 重新加载容器
#查看doker ip
docker inspect --format='{{.NetworkSettings.IPAddress}}' redis-master
~~~

#### docker 创建K8S的密钥

~~~shell
kubectl create secret docker-registry add-gs-secret --namespace=kube-add \
--docker-server=harbor.xxx.cn:1443/add-xxxway --docker-username=admin \
--docker-password='!@#' --docker-email=baru_sun@qq.com
~~~

~~~shell
#template.secret下使用密码
    spec:
      imagePullSecrets:
      - name: add-harbor-secret
~~~



#### docker运行nginx

~~~shell
 docker run \
  -d \
  -p 127.0.0.2:80:80 \
  -e PATH="/etc/" \
  --rm \
  --name mynginx \
  nginx
~~~

~~~shell
#docker daemon 文件
[root@bjb-falcon-053020 pods]# cat /etc/docker/daemon.json 
{
	"registry-mirrors": [
		"https://registry.docker-cn.com"
	],
   "max-concurrent-downloads": 10,
	"insecure-registries": [
		"172.21.52.62:5000"
	],
  "graph": "/data/docker"
}

~~~

#### docker打镜像

~~~shell
docker build -f Dockerfile.add -t harbor.xxx.cn:1443/add-registry/add:xxx_Build1634 .
~~~

#### docker 镜像迁移

~~~shell
#需求背景，主机B上下载不了一个镜像，，但是主机A上有这个镜像
#主机A上
# docker images  |grep dockerhub.azk8s.cn/jaegertra
dockerhub.azk8s.cn/jaegertracing/jaeger-agent             1.17.1                       6f5098716d02        3 months ago        25.2MB
# docker save -o jeager.tar dockerhub.azk8s.cn/jaegertracing/jaeger-agent:1.17.1
root@bjb-add-052045:~# ls
jeager.tar
#主机B上
[root@bjb-web-052015 xiaoxiao]# docker load -i jeager.tar 
a0500770559d: Loading layer [==================================================>]    236kB/236kB
134227daea79: Loading layer [==================================================>]  24.95MB/24.95MB
Loaded image: dockerhub.azk8s.cn/jaegertracing/jaeger-agent:1.17.1
[root@bjb-web-052015 xiaoxiao]# docker image ls
REPOSITORY                                               TAG                 IMAGE ID            CREATED             SIZE
harbor.voiceads.cn:1443/fax/logbeat                      0.9.11              b71f15dd43eb        30 hours ago        397MB
dockerhub.azk8s.cn/jaegertracing/jaeger-agent            1.17.1              6f5098716d02        3 months ago        25.2MB
172.21.52.62:5000/prom/node-exporter                     latest              188af75e2de0        2 years ago         22.9MB
172.21.52.62:5000/gcr.io/google_containers/pause-amd64   3.0                 f9d5de079539        5 years ago         240kB

~~~

#### docker 修改默认IP网段

修改 docker 配置 `/etc/docker/daemon.json` 文件，具体如下修改

~~~shell
# 添加下面配置
$ vim /etc/docker/daemon.json

{"bip": "10.50.0.1/16", "default-address-pools": [{"base": "10.51.0.1/16", "size": 24}]}

# 重启 docker 服务
$ systemctl restart docker
~~~

> 上面配置意思：设置 `docker0` 使用 `10.50.0.1/16` 网段，`docker0` 为 `10.50.0.1`。后面服务再创建地址池使用 `10.51.0.1/16` 网段范围划分，每个子网掩码划分为 `255.255.255.0`。

参考文档：[Docker 网络配置那些事](https://mp.weixin.qq.com/s/aTaD4xBUSFx_ZRiWlJjNmg)

#### Docker删除镜像registy的方法

~~~shell
Docker 仓库删除镜像的方法:


1、找出想要的镜像名称的tag
# curl 172.21.51.50:5000/v2/ubuntu/tags/list
{"name":"ubuntu","tags":["16.04_base","16.04-python2.7.13","16.04-jre1.8.0_131","16.04-pyenv","16.04-jre1.8_jce","16.04-python3.6.5"]}

2、拿到digest_hash参数

# curl  --header "Accept: application/vnd.docker.distribution.manifest.v2+json" -I -X GET http://172.21.51.50:5000/v2/ubuntu/manifests/16.04-python3.6.5
HTTP/1.1 200 OK
Content-Length: 1782
Content-Type: application/vnd.docker.distribution.manifest.v2+json
Docker-Content-Digest: sha256:8b1e9cd0ef601e055ae19d2e3d855cafc405eab22d3940ebdc745a330bfe7b3f
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:8b1e9cd0ef601e055ae19d2e3d855cafc405eab22d3940ebdc745a330bfe7b3f"
X-Content-Type-Options: nosniff
Date: Tue, 11 Dec 2018 01:24:09 GMT


3、删除镜像

curl -X DELETE http://172.21.51.50:5000/v2/ubuntu/manifests/sha256:8b1e9cd0ef601e055ae19d2e3d855cafc405eab22d3940ebdc745a330bfe7b3f



~~~

#### redhat&centos7 安装指定docker

~~~shell
#yum-utils可以理解为yum的扩展
yum install -y yum-utils
#添加阿里docker repo源
yum-config-manager --add-repo  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#yum makecache
#安装前查看所有版本，确定是否有自己需要的版本
yum list docker-ce --showduplicates
#安装指定版本
yum install -y docker-ce-19.03.4-3.el7
~~~



### dumpe2fs_查看文件系统块大小

~~~shell
dumpe2fs /dev/vdb1 |grep "Block size"
~~~



### dig命令

~~~shell
#查询域名
dig www.baidu.com
#指定解析服务器
dig www.baidu.com @10.255.255.88
#解析详情
dig www.baidu.com @114.114.114.114 +trace
~~~

###  dmesg命令

~~~shell
 #打印系统日志
 dmesg -T
~~~

### du 排查根磁盘空间

~~~shell
du -sh * --exclude=proc
~~~





## E

- [Table of Contents](#Table-of-Contents)

### ethool

~~~shell
##网卡多队列
ethtool -l eth0
##
设置网卡多队列
ethtool -L eth0 combined x
##查看网卡信息
ethtool eth0 
~~~

~~~shell
	[root@test ~]# ethtool -l eth0
	Channel parameters for eth0:
	Pre-set maximums:
	RX:		0
	TX:		0
	Other:		0
	Combined:	4      # 此行代表最多支持4个队列
	Current hardware settings:
	RX:		0
	TX:		0
	Other:		0
	Combined:	1      # 此行代表当前生效1个队列
	[root@test ~]# ethtool -L eth0 combined 4
	

# ethtool  enp175s0f1
Settings for enp175s0f1:
        Supported ports: [ FIBRE ]
        Supported link modes:   10000baseT/Full
        Supported pause frame use: Symmetric
        Supports auto-negotiation: No
        Advertised link modes:  Not reported
        Advertised pause frame use: No
        Advertised auto-negotiation: No
        Speed: 10000Mb/s
        Duplex: Full
        Port: Direct Attach Copper
        PHYAD: 0
        Transceiver: external
        Auto-negotiation: off
        Supports Wake-on: g
        Wake-on: g
        Current message level: 0x0000000f (15)
                               drv probe link timer
        Link detected: yes(查看是否连网线)
~~~

> ethtool 可以查看很多底层的物理层内容

参考文档：[京东云文档](https://docs.jdcloud.com/cn/virtual-machines/configurate-eni-multi-queue)

~~~shell
#查看网卡信息：传输速率、网线是否连接
ethtool bond0（网卡名）
~~~

### bond0配置

~~~shell
auto eno1
iface eno1 inet manual
bond-master bond0

auto eno2
iface eno2 inet manual
bond-master bond0

auto bond0
iface bond0 inet static
address 172.21.59.33
netmask 255.255.255.0
gateway 172.21.59.1
bond-slaves eno1 eno2
bond-mode 1
bond-miimon 100
~~~



## F

- [Table of Contents](#Table-of-Contents)

### fd_查看文件fd

~~~shell
#查看系统fd
cat /proc/fs/file-nr
#查看某进程的限制
cat /proc/XXX/limits
#某进程已经打开的fd
ls -l /proc/5454/fd/
~~~

参考文档：[Nginx: Too Many Open Files解决方案汇总](https://blog.csdn.net/jacson_bai/article/details/42171637)

## G

- [Table of Contents](#Table-of-Contents)

### grep命令

~~~shell
#不匹配带t的行
grep -v t test1
#匹配多个字段
grep -e t -e f test1 (含有t或者f的行)
#关键字附近几行
grep -A 5 后面5行  grep -B 5 前面5行
~~~

### git命令

~~~shell
git clone --depth=1  https://github.com/golang/net.git $GOPATH/src/golang.org/x/net
~~~

### go命令

~~~shell
##查看go环境变量
go env
~~~



## H

- [Table of Contents](#Table-of-Contents)

### httping

~~~shell
httpiing -S
dns解析时间、建立链接时间、send请求时间、处理请求和返回响应时间、连接断开时间
~~~



## I

- [Table of Contents](#Table-of-Contents)

### impi命令

系统不自带，需要安装

~~~shell
#安装
apt-get install ipmitool
#打印impi地址信息
ipmitool lan print 1
#启动模块：
modprobe ipmi_msghandler
modprobe ipmi_devintf
modprobe ipmi_si
modprobe ipmi_poweroff
modprobe ipmi_watchdog
#重启本地带外管理卡
ipmitool mc reset cold
~~~



### iptables命令

~~~shell
#查看
iptables -t nat -L -nv --line-numbers
#做端口转发
## 172.21.53.20:7000为后端服务器 172.16.59.106:10070为转发端口
iptables -t nat -A PREROUTING -d 172.16.59.106 -p tcp --dport 10070 -j DNAT --to 172.21.53.20:7000
iptables -A FORWARD -d 172.21.53.20 -p tcp --dport 7000 -j ACCEPT
iptables -t nat -A POSTROUTING -d  172.21.53.20  -p tcp --dport 7000 -j SNAT --to 172.16.59.106
iptables-save
#删除
iptables -t nat -D PREROUTING 118
iptables -t nat -D POSTROUTING 223
~~~

~~~shell
#设置tcp协商的mss
iptables -t mangle -I POSTROUTING -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1024
~~~

> 修改本机tcp的mss，会使本机和外界通信，发起tcp会话的时候，协商的mss是1024.

参考文档：[**由于TCPMSS配置过大导致上网慢**](https://support.huawei.com/enterprise/zh/knowledge/EKB1000052671)

### 修改IP重启networking不生效的问题

> Ubuntu 16.04 修改interfaces文件之后，重启网络服务之后，地址没有生效

~~~shell
ip addr flush dev ens33
ifdown ens33
ifup ens33
~~~



## J

- [Table of Contents](#Table-of-Contents)

### journalctl命令

~~~shell
#持续监控一个service的日志信息
journalctl -f -u kubelet.service
#查看整个service的日志信息
journalctl -u kubelet.service
~~~



### JDK

OpenJDK地址

~~~shell
https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_linux-x64_bin.tar.gz
~~~

#### JDK设置信息

~~~shell
#使用不同的JDK版本在于软链到什么文件夹
mkdir -p /usr/lib/jvm
wget -P /usr/lib/jvm/  http://172.21.52.62/jdk1.8.0_131.tar.gz
cd /usr/lib/jvm && tar -zxvf jdk1.8.0_131.tar.gz
ln -s jdk1.8.0_131 ./jdk
#JDK11
wget -P /usr/lib/jvm/  http://172.21.52.62/openjdk-11.0.2_linux-x64_bin.tar.gz
cd /usr/lib/jvm && tar -zxvf openjdk-11.0.2_linux-x64_bin.tar.gz
ln -s jdk-11.0.2 ./jdk

cat >> /etc/profile <<end
export LD_LIBRARY_PATH=./
export JAVA_HOME=/usr/lib/jvm/jdk
JRE_HOME=/usr/lib/jvm/jdk/jre
CLASSPATH=.:/usr/lib/jvm/jdk/lib:/usr/lib/jvm/jdk/lib
export PATH=/usr/lib/jvm/jdk/bin:$PATH:/opt/kubernetes/bin
end
~~~

#### 下载JDK

~~~shell
apt-get install -y openjdk-8-jdk
~~~



## K

- [Table of Contents](#Table-of-Contents)

### Kernel_Centos7内核操作

~~~shell
#查看可用内核
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
#查看当前内核
uname -r
#查看启动内核
grub2-editenv list
#修改启动内核
grub2-set-default 'CentOS Linux (3.10.0-327.el7.x86_64) 7 (Core)'
#查看安装了哪些内核
rpm -qa |grep kernel
#删除无用内核
yum remove kernel-3.10.0-327.el7.x86_64

~~~

## L

- [Table of Contents](#Table-of-Contents)

### lsof命令

~~~shell
#查看已经删除但是仍然被进程占用的进程
lsof -n|grep -i delete
~~~

参考文档：[lsof更多解释](<https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/lsof.html>)

## M

- [Table of Contents](#Table-of-Contents)

### mv

~~~shell
move -i  ：若目标文件已经存在，就会询问是否覆盖
~~~



### mysql命令

~~~shell
#连接
mysql -u root -p
#查看是否区分大小写
show global variables like '%lower_case%';
#修改用户密码
set password for 'root'@'localhost'=password('MYSQL123qwe?');
~~~



## N

- [Table of Contents](#Table-of-Contents)

### noload 查看网卡流量

~~~shell
nload eno3
~~~



### NC命令（监听端口）

~~~shell
#监听端口
nc -lp 8081

nc 127.0.0.1 9999 -e /bin/sh &
~~~

### NSCD服务

#### 作用

> ​	开启 nscd 的 hosts 缓存服务后，每次内部接口请求不会都发起 dns 解析请求，而是直接命中 nscd 缓存散列表，从而获取对应服务器 ip 地址，这样可以在大量内部接口请求时减少接口的响应时间

> **NOTE：**Linux系统不自带缓存

~~~shell
#清除缓存
nscd -i hosts
#查看统计信息
nscd -g
~~~

### NGINX

#### 代理的概述

> 所谓正向、反向就是代理的对象不同，正向代理的代理对象是客户端,反向代理的代理对象是服务端，代理服务器在客户端那边就是正向代理，代理服务器在原始服务器那边就是反向代理
>
> ​		用户的一次访问诉求中是否可以即经历的正向代理又经历的反向代理：假设我在公司的内网环境中访问不了百度，此时我可以通过一个代理(正向代理)服务器来访问百度，当我的代理服务器将我的请求转发给百度时，我的代理服务器对百度而言就是一个客户端，百度可能有一个前置的反向代理服务器，将我的请求转发给一个真正可以处理我的请求的应用服务器。那在这整个链路中就即经历的正向代理又经历的反向代理。



#### nginx站点直接返回200

~~~shell
server {
        listen 80;
        server_name silk1.voiceads.cn;
        #error_log logs/post_error.log;
        access_log logs/silk1_access.log;
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
                root   /var/www;
        }
	more_set_headers 'Content-Type: text/html; charset=utf-8';
        location / {
               return 200 "";
        }
        }
#以上是返回空字符串，也可以在双引号里加信息，返回给client
~~~

#### nginxlocal匹配顺序

~~~shell
nginx：
    1.local匹配优先级顺序：
    匹配顺序如下：
    (location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (location /)
    即（精确匹配）> (最长字符串匹配，但完全匹配) >（非正则匹配）>（正则匹配）>（最长字符串匹配，不完全匹配）>（location通配）
~~~

####  nginx 指令用途查询

[官网按指令字母顺序查询](http://nginx.org/en/docs/dirindex.html)

## O

- [Table of Contents](#Table-of-Contents)

### openssl 查看证书过期时间

~~~shell
openssl x509 -text -in /opt/kubernetes/ssl/ca.pem |grep Not
~~~



## P

- [Table of Contents](#Table-of-Contents)

### perf

> 排查cpu百分百的时候，可以使用perf排查，需要分析查找到程序百分比高的热点代码片段，这便需要使用 perf record 记录单个函数级别的统计信息，并使用 perf report 来显示统计结果；

~~~shell
#安装
apt install linux-tools-common
apt install linux-tools-4.4.0-145-generic（根据内核版本来）
# 
perf top 
#
perf top -p xxxpid
#
perf record -e cpu-clock -g -p 2548
-g 选项是告诉perf record额外记录函数的调用关系
-e cpu-clock 指perf record监控的指标为cpu周期
-p 指定需要record的进程pid
#程序运行完之后，perf record会生成一个名为perf.data的文件，如果之前已有，那么之前的perf.data文件会被覆盖
#得这个perf.data文件之后，就需要perf report工具进行查看
perf report -i perf.data
-i 指定要查看的文件
(还没看懂结果)

~~~

~~~shell
# 火焰图制作
1、Flame Graph项目位于GitHub上：https://github.com/brendangregg/FlameGraph

2、可以用git将其clone下来：git clone https://github.com/brendangregg/FlameGraph.git
我们以perf为例，看一下flamegraph的使用方法：
1、第一步
$sudo perf record -e cpu-clock -g -p 28591
Ctrl+c结束执行后，在当前目录下会生成采样数据perf.data.
2、第二步
用perf script工具对perf.data进行解析
perf script -i perf.data &> perf.unfold
3、第三步
将perf.unfold中的符号进行折叠：
#./stackcollapse-perf.pl perf.unfold &> perf.folded
4、最后生成svg图：
./flamegraph.pl perf.folded > perf.svg
该文件可以使用浏览器查看
~~~

~~~shell
#java程序火焰图制作
git clone https://github.com/jvm-profiling-tools/async-profiler
yum install gcc-c++ -y
#Ubuntu如下安装 gcc+
apt-get install build-essential

export JAVA_HOME=/usr/local/jdk1.8.0_241/(针对自己系统的java目录修改)
make
使用
录制
./profiler.sh -d 60 -o collapsed -f /root/test_01.txt 2879 (java进程id，会包含下面线程)
./FlameGraph/flamegraph.pl test_01.txt > test_01.svg
~~~

参考资料： [perf + 火焰图分析程序性能](https://www.cnblogs.com/happyliu/p/6142929.html)

​					[如何读懂火焰图](http://www.ruanyifeng.com/blog/2017/09/flame-graph.html)

###  排查javacpu利用率问题

~~~shell
方法一：
转载：http://www.linuxhot.com/java-cpu-used-high.html

1.jps 获取Java进程的PID。

2.jstack pid >> java.txt 导出CPU占用高进程的线程栈。

3.top -H -p PID 查看对应进程的哪个线程占用CPU过高。

4.echo “obase=16; PID” | bc 将线程的PID转换为16进制,大写转换为小写。

5.在第二步导出的Java.txt中查找转换成为16进制的线程PID。找到对应的线程栈。

6.分析负载高的线程栈都是什么业务操作。优化程序并处理问题。

 

方法二：
1.使用top 定位到占用CPU高的进程PID

top 

通过ps aux | grep PID命令

2.获取线程信息，并找到占用CPU高的线程

ps -mp pid -o THREAD,tid,time | sort -rn 

3.将需要的线程ID转换为16进制格式

printf "%x\n" tid

4.打印线程的堆栈信息

jstack pid |grep tid -A 30
~~~

参考：[排查Java高cpu占用原因](https://www.cnblogs.com/cnndevelop/p/11091813.html)

### PS

~~~shell
#查守护进程
ps -eo ppid,pid,sid,stat,tty,comm  | awk '{ if ($2 == $3 && $5 == "?") {print $0}; }'
~~~

~~~shell
#查看该进程开了多少线程
ps -huH -p 29202  |wc -l
#查看该进程线程的内存使用情况
top -Hp 181310
~~~

## Q

- [Table of Contents](#Table-of-Contents)



## R

- [Table of Contents](#Table-of-Contents)

### route_linux路由

~~~shell
添加
route add -net 172.22.241.0 netmask 255.255.255.0 gw 172.21.52.1
删除
route del -net 172.22.241.0 netmask 255.255.255.0 gw 172.21.52.1
#删除网关
route delete -net 0.0.0.0
#添加网关
route add -net 0.0.0.0 netmask 0.0.0.0 gw 114.118.65.1
~~~

### rsync

~~~shell
#同步文件
CURRENT_DIR=$(cd "$(dirname "$0")";pwd)
rsync -av $1 -e 'ssh -p 22' 10.249.1.4:$CURRENT_DIR/
~~~



### redis命令

#### 扩容

~~~shell
./redis-cli --cluster add-node 192.168.76.112:8005  192.168.76.111:8003
./redis-cli --cluster reshard 192.168.76.111:8003
./redis-cli --cluster add-node 192.168.76.112:8006 192.168.76.112:8005 --cluster-slave --cluster-master-id de23774ef05d33da4e9f6aa8803debe139690e41
~~~

#### 缩容

~~~shell
./redis-cli --cluster del-node 192.168.76.112:8006 ba71b5c61e9f40e1211569f36fd8e309dd8a6987
./redis-cli --cluster reshard 192.168.76.112:8005
./redis-cli --cluster del-node 192.168.76.112:8005 de23774ef05d33da4e9f6aa8803debe139690e41 
~~~

~~~shell
#集群信息查看
./redis-cli --cluster info 192.168.76.112:8005
#平衡槽数量
./redis-cli --cluster rebalance 192.168.76.111:8004
~~~

**note：**node失联是否会影响到集群的正常工作，关键在于该失联的node是否有管理的slot，如果有则影响；

​	 尽量避免master和slave在同一台集群上，否则

常用命令：

~~~shell
#查看配置
config get maxmemory
#修改配置
config set maxmemory 30G
#保存配置至配置文件
config rewrite
#将一个slave 切为主master
redis-cli -h <从节点ip> -p <从节点端口号> slaveof no one
~~~



## S

- [Table of Contents](#Table-of-Contents)

### ssh免密登陆

~~~shell
ssh-keygen -t rsa
ssh-copy-id root@10.10.5.3
#或者
ssh-keygen -t rsa
cat ~/.ssh/id_*.pub|ssh root@172.17.173.67 'cat>>.ssh/authorized_keys'
~~~

~~~shell
#移除免密认证或者认证信息
 ssh-keygen -f "/root/.ssh/known_hosts" -R 172.31.131.174
~~~

### ssh设置白名单

~~~shell
#修改sshd_confi给文件，尾部添加如下 ：
# Example of overriding settings on a per-user basis
#Match User anoncvs
#       X11Forwarding no
#       AllowTcpForwarding no
#       PermitTTY no
#       ForceCommand cvs server
Match Address 172.21.52.54
        AllowUsers mysftp
        AllowUsers UnionPay
~~~

参考链接：[ssh登陆限制](http://www.361way.com/ssh-root-login-limit/5813.html)

~~~shell
#debug ssh登陆信息
ssh -v
~~~

### sed命令

sed替换：

~~~shell
sed -i 's/service_task_num=16/service_task_num=4/g' conf/add.conf
grep service_task_num conf/add.conf
~~~

sed行插入

~~~shell
#行下插入
sed -i '/service_task_num/a DISPATCHER_SIZE=4'  conf/add.conf
grep DISPATCHER_SIZE=4 conf/add.conf
#行上插入
sed -i '/service_task_num/i DISPATCHER_SIZE=4'  conf/add.conf
sed -i '/^modprobe/i\/opt/open-falcon/agent/control start' /etc/rc.local（转义插入）
sed -i '/#save 900/i save \"\"' 800*/redis.conf （特殊字符“”插入）
~~~

sed尾部追加

~~~shell
#修改
sed -i 's/$/&:8003/g' reids-node
#不修改文件内容，并输出
sed  's/$/&:8003/g' reids-node
~~~

sed匹配到一行的关键字，并替换该行

~~~shell
sed -i 's/^.*aaaaaaaa.*$/bbbbbbbbbbbbbbbbbbbb/' add.conf.bak
~~~

sed 在匹配到一行的隔一行插入

~~~shell
sed '/ServiceAccount/N;N;a \ \ namespace\:test' test.yaml
~~~



### sort命令

~~~shell
#指定关键字排序
sort -t ':' -k 2 /etc/passwd
#按数值大小排序
sort -nr /etc/passwd （-r 从大到小）
#-M 按照月份排序
~~~

### systemctl 的service文件解析

~~~shell
#type字段
Type字段定义启动类型。它可以设置的值如下。

simple（默认值）：ExecStart字段启动的进程为主进程
forking：ExecStart字段将以fork()方式启动，此时父进程将会退出，子进程将成为主进程
oneshot：类似于simple，但只执行一次，Systemd 会等它执行完，才启动其他服务
dbus：类似于simple，但会等待 D-Bus 信号后启动
notify：类似于simple，启动结束后会发出通知信号，然后 Systemd 再启动其他服务
idle：类似于simple，但是要等到其他任务都执行完，才会启动该服务。一种使用场合是为让该服务的输出，不与其他服务的输出相混合
~~~

### systemctl设置ubuntu启动

~~~shell
桌面：systemctl set-default graphical.target
命令行：systemctl set-default multi-user.target
~~~

### 输出重定向

~~~shell
 >> ../tmp/tmp.txt 2>&1
~~~

### su

~~~shell
#su 带密码交互运行
expect -c 'spawn su baru;expect "Password";send "123qwe\r";interact'
~~~







## T

- [Table of Contents](#Table-of-Contents)

### timedatectl 查看时区

~~~shell
timedatectl status
      Local time: Wed 2020-06-17 14:34:24 CST
  Universal time: Wed 2020-06-17 06:34:24 UTC
        RTC time: Wed 2020-06-17 06:34:24
       Time zone: Asia/Shanghai (CST, +0800)
 Network time on: yes
NTP synchronized: yes
 RTC in local TZ: no

~~~



### tcpdump命令

~~~shell
tcpdump -vv -nn -i eth0  tcp port 30082
tcpdump -i eth0 -s 100 -c 500000 -w nginx-directly-eth0-keda.pcap
tcpdump -i any host 106.39.171.38 -nnevv -c 10000 -w tmp.pcap
tcpdump -nn -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-fin)!=0' and host 123.125.23.101 and tcp port 80
#抓http包
tcpdump -i bond0 port 8080 -A -s 0 -c 10
~~~



### Wirsharek 过滤条件

~~~shell
tcp.flags.reset == 1 && ( ip.addr == 10.10.64.35 || ip.addr == 10.10.64.8 || ip.addr == 10.10.64.39 || ip.addr == 10.10.64.22 )
~~~

~~~shell
# 在wireshark众多包中快速过滤发了sync包但是没有回sync+ack的包
tcp.flags.syn == 1 && tcp.analysis.retransmission
~~~



## U

- [Table of Contents](#Table-of-Contents)

## V

- [Table of Contents](#Table-of-Contents)

### vim

~~~shell
# vim 直接定位到哪一行
vim ifly_cpcc_ad_basic_nocomment.sql  +19
~~~

~~~shell
#vim 二进制打开文件
vim -b filename
然后
:%!xxd
~~~



## W

- [Table of Contents](#Table-of-Contents)

### wget命令

~~~shell
#下载到指定位置
wget -O
~~~

### windows terminl 快捷键

~~~shell
左右分屏：Alt+Shift+=

上下分屏：Alt+Shift+-

取消分屏：Ctrl+Shift+w
~~~



## X

- [Table of Contents](#Table-of-Contents)

### xargs

~~~shell
xargs -i kubectl label node {}  kubernetes.io/role=add-node
~~~



## Y

- [Table of Contents](#Table-of-Contents)

### yum

~~~shell
#yum 锁定软件版本
#安装hold扩展
yum install yum-plugin-versionlock.noarch -y
#锁定docker（示例）
yum versionlock docker-ce
#可以使用delete来解锁：
yum versionlock delete docker-ce

#centos
查找
yum list installed
rpm -qa
yum list installed |grep vsftpd
卸载
yum -y remove vsftpd
~~~



## Z

- [Table of Contents](#Table-of-Contents)

## Kubernetes

- [Table of Contents](#Table-of-Contents)

### Kubectl 自动补全

~~~shell
source <(kubectl completion bash) # 在 bash 中设置当前 shell 的自动补全，要先安装 bash-completion 包。
echo "source <(kubectl completion bash)" >> ~/.bashrc # 在您的 bash shell 中永久的添加自动补全
~~~



### 获取pod信息

~~~shell
#podIP
kubectl  get pod add-6xvhz -o go-template={{.status.podIP}}
#pod的状态
kubectl  get pod add-6xvhz -o go-template={{.status.phase}}
#有6种：（Pending、Running、Succeeded、Failed、Unknown）
kubectl  get pod add-6xvhz -o go-template={{.status}}
~~~

### 给podcopy东西：

~~~shell
#往外面copy
kubectl -n kube-add cp add-hcxdj:/add/profile.svg -c add02 .
#往里面copy
kubectl -n kube-add cp xx add-ds12-n9tzs:/tmp/ -c add01
~~~



### 设置pod镜像

~~~shell
kubectl set image pod add-x6l67  add01=172.21.52.62:5000/add:addserver_Build1276 add02=172.21.52.62:5000/add:addserver_Build1276
~~~

~~~shell
#添加kubernetes源
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
#阿里源
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
#
apt-get update
apt-get install -y kubelet kubeadm kubectl
 kubeadm init --ignore-preflight-errors=Swap
~~~

### 生成仓库secret

~~~shell
kubectl create secret docker-registry stress-testing-secret --namespace=stress-testing \
--docker-server=harbor.xxx.cn:1443/stress-testing --docker-username=user_name \
--docker-password='xxxxx' --docker-email=barusun@qq.com
~~~

### docker 打镜像

~~~shell
docker build -f Dockerfile.add -t harbor.xxx.cn:1443/add-registry/add:xxx_Build1634 .
~~~

### 给节点打角色标签

~~~shell
kubectl label node k8s-m1 kubernetes.io/role=master
~~~

###  进入容器

~~~shell
kubectl exec -i -t -n kube-add add-ds12-n4m42 -c add01 "--" sh -c "clear; (bash || ash || sh)"
~~~

### 根据进程id获取podname脚本

~~~shell
需要先安装：jq
apt-get install jq -y
~~~

~~~shell
#!/usr/bin/env bash

Check_jq() {
  which jq &> /dev/null
  if [ $? != 0 ];then
    echo -e "\033[32;32m 系统没有安装 jq 命令，请参考下面命令安装！  \033[0m \n"
    echo -e "\033[32;32m Centos 或者 RedHat 请使用命令 yum install jq -y 安装 \033[0m"
    echo -e "\033[32;32m Ubuntu 或者 Debian 请使用命令 apt-get install jq -y 安装 \033[0m"
    exit 1
  fi
}

Pod_name_info() {
  CID=`cat /proc/${pid}/cgroup | head -1 | awk -F '/' '{print $5}'`
  CID=$(echo ${CID:0:8})
  docker inspect $CID | jq '.[0].Config.Labels."io.kubernetes.pod.name"'
}

pid=$1
Check_jq
Pod_name_info
~~~

~~~shell
#执行：
 ./pod_name.sh 146762
"add-xxx-hcxdj"
~~~

参考文档：[根据 PID 获取 K8S Pod名称 - 反之 POD名称 获取 PID](https://mp.weixin.qq.com/s/HF5rzr5fULiMWq1NPe780g)

###  flannel 网络工作原理分析

1[Kubernetes网络分析之Flannel](https://www.kubernetes.org.cn/4887.html)

2[Kubernetes中的网络解析——以flannel为例](https://jimmysong.io/kubernetes-handbook/concepts/flannel.html)



## 组合命令

- [Table of Contents](#Table-of-Contents)

### IP获取

~~~shell
ifconfig | grep "inet addr" | cut -d":" -f2 | cut -d" " -f1 | head -1
~~~

### 查看本机出口IP

~~~shell
curl ifconfig.me
curl cip.cc
~~~

### 查看TCP连接信息

~~~shell
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
~~~

### 批量删除进程

~~~shell
ps aux|grep mock|grep -v grep|cut -c 9-15|xargs kill -15
~~~



### for循环

~~~shell
for((i=0;i<3;i++)); do touch "test ${i}.log";done
~~~



## 脚本

- [Table of Contents](#Table-of-Contents)

### 服务监测脚本

~~~shell
#!/bin/bash
A=`ps -C nginx --no-header | wc -l`
if [ $A -eq 0 ];then
    /usr/local/openresty/nginx/sbin/nginx #尝试重新启动nginx
    sleep 2  #睡眠2秒
    if [ `ps -C nginx --no-header | wc -l` -eq 0 ];then
        killall keepalived #启动失败，将keepalived服务杀死。将vip漂移到其它备份节点
    fi
fi
~~~



### ssh免密登陆

~~~shell
#!/bin/bash

passwd="xxxxx"


smartssh () {
    expect -c "set timeout 1;
                spawn ssh root@$1;
                expect  *assword* 
                send -- ${passwd}\r;
	       interact
                "
    #return $?
}

read -p "Please import your dest sesrvier :" tmp_num
ip=10.248.$tmp_num
if ping -W 1 $ip -c 2 &>/dev/null; then
  smartssh $ip
else
  echo -e "\n\033[33;40;5;1m The Host unreachable,Please Check; Script Exit Now...\033[0m\n"
  echo -e "\n\033[33;40;5;1m ping检测失败，主机不可达，请检查你的IP...\033[0m\n"
  exit 0
fi
# 从键盘输入IP，先进行联通性检查，然后免密登陆
~~~



~~~shell
#!/usr/bin/expect

set timeout 30
spawn ssh xxx@172.16.12.200
expect "password"
send "x123qwe\r"

interact
~~~

### 批量操作Kubernetes pod

~~~shell
#!/bin/bash
#auth: xiaoxiao
#time: 20190911
shell_folder=$(cd "$(dirname "$0")";pwd)
i=1
#echo $shell_folder
kubectl get pod -o wide|grep test-ds |awk '{print $1}' >$shell_folder/podname.txt
for podname in $(cat $shell_folder/podname.txt)
do  
  #echo $podname
  sleep 1
  echo 
  echo $podname
    if [ $i -le 3 ]
    then
      kubectl exec $podname -c add01 -- bash -c "/bin/echo '172.22.241.41 mock.service.cn' >>/etc/hosts"
      sleep 1
      kubectl exec $podname -c add02 -- bash -c  "cat /etc/hosts"
    else
      kubectl exec $podname -c add03 -- bash -c "/bin/echo '172.22.241.41 mock.service.cn' >>/etc/hosts"
      sleep 1
      kubectl exec $podname -c add04 -- bash -c  "cat /etc/hosts"
    fi 
  let i++ 
done
~~~



### 获取当前目录：

> ~~~shell
>shell_folder=$(cd "$(dirname "$0")";pwd)
> ~~~

### 节点系统给信息收集脚本

~~~shell
#!/bin/bash

function ALL {
ss -ant | awk '{++s[$1]} END {for(k in s) print k,s[k]}'
}

softusage=`mpstat -P 12 1 1 | sed -n 4p | awk '{print int($9)}'`
threshold=25

if [ $softusage -gt $threshold ]; then
filename=`date +%Y-%m-%d-%T`
echo "checked softirq"$softusage >> $filename.log
count=10
for i in $count; do 
date +"%T.%6N" >> $filename.log

ALL >> $filename.log
ss -it >> $filename.log
nstat >> $filename.log
netstat -s >> $filename.log
cat /proc/softirqs >> $filename.log
cat /proc/interrupts >> $filename.log

sleep 1
date +"%T.%6N" >> $filename.log
ALL >> $filename.log
ss -it >> $filename.log
nstat >> $filename.log
netstat -s >> $filename.log
cat /proc/softirqs >> $filename.log
cat /proc/interrupts >> $filename.log

top -b -n 3 >> $filename.log
mpstat -P ALL 1 3 >> $filename.log
pidstat -t 1 3 >> $filename.log
sar -n DEV 1 3 >> $filename.log
sar -n EDEV 1 3 >> $filename.log
process=`ps -ef | grep jdk8/bin/java | grep -v grep | awk '{print $2}'`
for id in $process
do
echo process:$id >> $filename.log
ps huH p $id | wc -l >> $filename.log
#cat /proc/$id/status
top -b -H -p $id -n 3 >> $filename.log
done
sleep 1
done

softirq=`ps -ef | grep ksoftirqd/12 | grep -v grep | awk '{print $2}'`
perf record -a -g -p $softirq -o $filename.softirq.data -- sleep 30
perf record -c 12 -g -o $filename.cpu.data -- sleep 30

#else

#filename1=`date +%Y-%m-%d-%T`
#echo "checked softirq"$softusage >> $filename1.nochecklog
fi
~~~



### 遍历redis节点

~~~shell
#!/bin/bash
#auther: xiaoxiao
#time: 20190911
#description: Traversing the redis node
dir=/home/xiaoxiao
rediscli_dir=/usr/local/redis-cluster/8003
list="8003 8004 8005 8006"
for ip in $(cat $dir/node.txt)
do
  echo
  echo $ip:
  for node in $list
  do
    echo "the information of redis node: $ip:$node"
    $rediscli_dir/redis-cli -h $ip -p $node config set save ""
    sleep 1
    $rediscli_dir/redis-cli -h $ip -p $node config get save
  done
  sleep 1  
done
~~~

### 添加端口映射

~~~shell
#!/bin/bash
#pro='tcp'
#srcip='172.16.59.106'
export srcip="172.16.59.106"
echo -e "\033[49;32;1m $date\033[49;35;1m添加防火墙端口映射\033[0m "
read -p "(请输入本机的映射端口)": dport
echo "..."
read -p "(请输入内网目的主机的IP)": in_host
echo "..."
read -p "(请输入内网目的主机的端口号)": in_port
iptables -t nat -A PREROUTING  -p tcp -d $srcip --dport $dport -j DNAT --to $in_host:$in_port
iptables -t nat -A POSTROUTING -p tcp -d $in_host --dport $in_port -j SNAT --to $srcip
sleep 2
echo -e "\033[49;33;1mplease wait...\033[49;33;1m$srcip:$dport->$in_host:$in_port added\033[0m "
sleep 2
iptables -t nat -L -n --line-number | grep $dport
sleep 2
echo "iptables saving......"
sleep 2
iptables-save > /dev/null
echo -e "\033[49;32;1msave success!\033[0m "

~~~

###  获取解析一个域名所花的时间

~~~~shell
curl -o /dev/null -s -w time_namelookup:"\t"%{time_namelookup}"\n"time_connect:"\t\t"%{time_connect}"\n"time_appconnect:"\t"%{time_appconnect}"\n"time_pretransfer:"\t"%{time_pretransfer}"\n"time_starttransfer:"\t"%{time_starttransfer}"\n"time_total:"\t\t"%{time_total}"\n"time_redirect:"\t\t"%{time_redirect}"\n" https://sdk.voiceads.cn/​
~~~~

~~~shell
#输出
time_namelookup:        2.511462
time_connect:           2.532619
time_appconnect:        2.586215
time_pretransfer:       2.586290
time_starttransfer:     2.607437
time_total:             2.607498
time_redirect:          0.000000
~~~



> 段含义:
>
> time_namelookup: 0.004	dns查询时间：4ms
> time_connect: 0.016	建链时间：12ms
> time_appconnect: 0.109	ssl/tls握手时间：93ms
> time_pretransfer: 0.109
> time_starttransfer: 0.123 开始传输到server开始返回耗时：14ms
> time_total: 0.123 总耗时：123ms
> time_redirect: 0.000 重定向耗时：0ms
>
> 当然还有更多的字段可以通过man curl的方式查看

### crontab自动清理日志

> nacaos的access日志，在当前版本不支持自动清理，这是因为nacos 用了 springboot, 间接用了 tomcat, 又开启了 access log, 于是就打出来了。要么关掉，要么使用crontab自动清理。该脚本放在/etc/cron.daily/ 目录下。cron.daliy执行时间如下
>
> ~~~shell
> # cat /etc/crontab
> # /etc/crontab: system-wide crontab
> # Unlike any other crontab you don't have to run the `crontab'
> # command to install the new version when you edit this file
> # and files in /etc/cron.d. These files also have username fields,
> # that none of the other crontabs do.
> 
> SHELL=/bin/sh
> PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
> 
> # m h dom mon dow user  command
> 17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
> 25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
> 47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
> 52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
> #
> ~~~
>
> 

~~~shell
#nacosDelAccessLogs.sh
#!/bin/bash

logFile="/data/nacos/logs/nacos_del_access.log"
# 保留7天日志
date=`date -d "$date -7 day" +"%Y-%m-%d"`
# 具体位置可调整
delFilePath="/logs/access_log.${date}.log"

if [ ! -f "${logFile}" ];then
	echo 'access log文件打印日志频繁. /etc/cron.daily/nacosDelAccessLogs.sh 会定时删除access日志文件' >>${logFile}
fi
# 日志文件存在， 则删除
if [  -f "${delFilePath}" ];then
	rm -rf ${delFilePath}
	curDate=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
	echo '['${curDate}'] 删除文件'${delFilePath} >>${logFile}
fi
~~~

参考文档：[Ubuntu cron 定时执行任务](jianshu.com/p/d6d8d9f7f60c);

​					[nacos access log日志占用磁盘](https://blog.csdn.net/wtzvae/article/details/107212870)

​					[Nacos 官方文档](https://nacos.io/zh-cn/docs/faq.html)

## Prometheus

### 启动

~~~shell
docker run --name prometheus -d -p 9090:9090 -v /data/prometheus-data:/prometheus-data -v /data/prometheus-data/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus --config.file=/etc/prometheus/prometheus.yml
~~~

~~~shell
#热加载配置文件
kill -HUP pid
curl -X POST http://IP/-/reload
~~~



### 启动 grafana

~~~shell
cat start_grafana.sh 
docker run \
  -d \
  -p 3000:3000 \
  --name=grafana \
  grafana/grafana
#安装指定版本grafana
docker run \
  -d \
  -p 3000:3000 \
  --name=grafana \
  grafana/grafana:4.3.2
~~~

安装Open-Falcon插件

~~~shell
#以root方式启动grafana
docker exec -it -u root b0e3350b61a0 bash
#更新grafana
apt-get update
#
apt-get install git
#
cd /var/lib/grafana/plugins/
#
git clone https://github.com/open-falcon/grafana-openfalcon-datasource
#注意容器内/etc/grafana/grafana.ini配置文件的路径
#退出重启docker
docker restart grafana
~~~

配置grafana邮件告警

~~~shell
#编辑/etc/grafana/grafana.ini文件，修改如下 
[smtp]
enabled = true
host = mail.iflytek.com:465
user = adfalcon
# If the password contains # or ; you have to wrap it with trippel quotes. Ex """#password;"""
password = """123qwe"""
;cert_file =
;key_file =
skip_verify = true
from_address = 122@123.com
from_name = JD_Grafana
~~~



### GIT

~~~shell
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/Barusun/xiaoxiao.github.io.git
git push -u origin master


git remote add origin https://github.com/Barusun/xiaoxiao.github.io.git
git push -u origin master
~~~



