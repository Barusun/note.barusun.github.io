# Problem Record



## Linux 

### nf_conntrack 表满

1. 问题表现：ping某公网入口IP有丢包现象

   ~~~shell
   ~# ping xxx.xxx.xxx.xxx
   PING xxx.xxx.xxx.xxx (xxx.xxx.xxx.xxx) 56(84) bytes of data.
   64 bytes from xxx.xxx.xxx.xxx: icmp_seq=12 ttl=54 time=2.82 ms
   64 bytes from xxx.xxx.xxx.xxx: icmp_seq=13 ttl=54 time=2.60 ms
   64 bytes from xxx.xxx.xxx.xxx: icmp_seq=14 ttl=54 time=2.81 ms
   64 bytes from xxx.xxx.xxx.xxx: icmp_seq=15 ttl=54 time=2.61 ms
   64 bytes from xxx.xxx.xxx.xxx: icmp_seq=16 ttl=54 time=2.66 ms
   64 bytes from xxx.xxx.xxx.xxx: icmp_seq=17 ttl=54 time=2.58 ms
   64 bytes from xxx.xxx.xxx.xxx: icmp_seq=18 ttl=54 time=2.62 ms
   ^C
   --- xxx.xxx.xxx.xxx ping statistics ---
   18 packets transmitted, 7 received, 61% packet loss, time 17096ms
   ~~~

2. 问题排查

   1. 入口服务器上有docker（docker会用到nf_conntrack表），一般情况下，入口的nginx机器在高并发的情况下，规范的时候会在开始启动的时候禁止加载：nf_conntrack模块，但是使用iptables或者docker的时候，会加载nf_conntrack模块。

      ~~~shell
      # systemctl status docker
      ● docker.service - Docker Application Container Engine
         Loaded: loaded (/lib/systemd/system/docker.service; disabled; vendor preset: enabled)
         Active: inactive (dead) since Wed 2020-10-14 09:34:06 CST; 15min ago
           Docs: https://docs.docker.com
       Main PID: 17907 (code=exited, status=0/SUCCESS)
      
      Oct 14 09:34:06 xxx.xxx.xxx systemd[1]: Stopping Docker Application Container Engine...
      Oct 14 09:34:06 xxx.xxx.xxx dockerd[17907]: time="2020-10-14T09:34:06.164540011+08:00" level=info msg="Processing signal 'terminated'"
      Oct 14 09:34:06 xxx.xxx.xxx dockerd[17907]: time="2020-10-14T09:34:06.165200990+08:00" level=info msg="stopping event stream following graceful shutdown" error="<nil>" mod
      Oct 14 09:34:06 xxx.xxx.xxx dockerd[17907]: time="2020-10-14T09:34:06.165572503+08:00" level=info msg="Daemon shutdown complete"
      Oct 14 09:34:06 xxx.xxx.xxx dockerd[17907]: time="2020-10-14T09:34:06.165619985+08:00" level=info msg="stopping event stream following graceful shutdown" error="context ca
      Oct 14 09:34:06 xxx.xxx.xxx systemd[1]: Stopped Docker Application Container Engine.
      ~~~

      > 之前是running状态，让我手动停掉了。

   2. 查看dmesg日志，检查nf_conntrack表被打满

      ~~~shell
      # dmesg -T |more
      [Wed Oct 14 09:00:18 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:18 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:18 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:18 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:18 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] net_ratelimit: 24072 callbacks suppressed
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:00:23 2020] nf_conntrack: table full, dropping packet
      ~~~

      最后时间：

      ~~~shell
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:29 2020] nf_conntrack: table full, dropping packet
      [Wed Oct 14 09:33:41 2020] nr_pdflush_threads exported in /proc is scheduled for removal
      ~~~

      > 公网入口丢包，之前遇到过两次，一个是nf_conntrack表满，一个是网卡打满。

3. 问题解决

   1. 卸载docker

      ~~~shell
      apt-get -y remove docker
      apt-get -y autoremove docker
      apt-get -y purge docker
      ~~~

   2. 增大nf_conntrack限制

   ~~~shell
   #编辑/etc/sysctl.conf文件
   net.nf_conntrack_max = 2147483647
   #使之生效：
   sysctl -p
   ~~~

   3. 查看

   ~~~shell
   #查看当前nf_conntrack使用大小:
   #sysctl net.netfilter.nf_conntrack_count
   net.netfilter.nf_conntrack_count = 1019770
   #查看当前nf_conntrack使用最带值:
   # sysctl net.netfilter.nf_conntrack_max
   net.netfilter.nf_conntrack_max = 2147483647
   ~~~

   > 还有一种问题解决办法，就是卸载nf_conntrack内核模块

4. 移除nf_conntrack模块

   ```shell
   #编辑：/boot/grub/grub.cfg，添加模块加载黑名单
   modprobe.blacklist=iptable_nat,nf_conntrack
   #备份iptables模块（iptables使用的话，会自动加载nf_conntrack）
   mv /lib/modules/4.4.0-62-generic/kernel/net/ipv4/netfilter/ip_tables.ko /lib/modules/4.4.0-62-generic/kernel/net/ipv4/netfilter/ip_tables.ko.bak
   ```

### 

~~~shell
* soft nofile 655365
* hard nofile 655365
* soft nproc 1030993
* hard nproc 1030993
* soft  memlock  unlimited
* hard memlock  unlimited

~~~



### xxx云入口