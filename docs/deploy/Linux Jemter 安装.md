### Linux Jemter 安装

[TOC]

## 环境准备

>  部署节点：10.100.84.18

### JDK环境检查

~~~shell
~# java -version
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
~~~

### Jmeter安装

1. 下载文件

~~~shell
mkdir /usr/local/jmeter/
cd /usr/local/jmeter/
wget http://mirrors.tuna.tsinghua.edu.cn/apache//jmeter/binaries/apache-jmeter-5.1.1.tgz
~~~



2. 解压文件：

~~~shell
cd /usr/local/jmeter/
tar -zxvf apache-jmeter-5.1.1.tgz 
~~~

3. 设置环境变量

~~~shell
vim /etc/profile
## 追加以下配置
export JMETER=/usr/local/jmeter/apache-jmeter-5.1.1
export CLASSPATH=${JMETER}/lib/ext/ApacheJMeter_core.jar:${JMETER}/lib/jorphan.jar:$JMETER/lib/logkit-2.0.jar:${CLASSPATH}
export PATH=${JMETER}/bin/:${PATH}
## 使环境变量生效
source /etc/profile
~~~

4. 验证

~~~shell
jmeter -v
##输出
Apr 02, 2019 10:05:55 AM java.util.prefs.FileSystemPreferences$1 run
INFO: Created user preferences directory.
    _    ____   _    ____ _   _ _____       _ __  __ _____ _____ _____ ____     
   / \  |  _ \ / \  / ___| | | | ____|     | |  \/  | ____|_   _| ____|  _ \   
  / _ \ | |_) / _ \| |   | |_| |  _|    _  | | |\/| |  _|   | | |  _| | |_) | 
 / ___ \|  __/ ___ \ |___|  _  | |___  | |_| | |  | | |___  | | | |___|  _ <  
/_/   \_\_| /_/   \_\____|_| |_|_____|  \___/|_|  |_|_____| |_| |_____|_| \_\ 5.1.1 r1855137  

Copyright (c) 1999-2019 The Apache Software Foundation

~~~

