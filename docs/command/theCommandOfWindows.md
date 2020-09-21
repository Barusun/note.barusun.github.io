# windows 端口转发

> windows server 12 r2 实践

> cmd命令行操作，需要adminstrator用户已管理员身份执行

~~~ powershell
 命令查看
 netsh interface portproxy show all
 添加转发
 netsh interface portproxy add v4tov4 listenaddress=192.168.0.101 listenport=7001 connectaddress=192.168.0.80 connectport=8080
                                                    服务器A，本机IP            本机端口             服务器B，远端IP            远端业务端口        
 删除转发记录
 netsh interface portproxy delete v4tov4 listenaddress=192.168.0.101 listenport=7001
 重启端口转发策略
 netsh interface portproxy reset
~~~

参考文档： [百度经验](https://jingyan.baidu.com/article/7c6fb428add0e7c0652c9072.html)

