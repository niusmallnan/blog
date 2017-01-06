+++
images = [
]
tags = [
    "Rancher",
    "Rancher v1.2"
]
date = "2017-01-06T15:12:10+08:00"
title = "Rancher-LB开启访问日志"
slug = "rancher-lb-logs"
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""

+++
拯救一脸懵逼
<!--more-->
### 引言
在Rancher v1.2之前的版本，Rancher LB也就是Haproxy都是放在agent/instance内的，由于这个镜像本身要支持很多功能，
所以要开启Haproxy的访问日志比较麻烦。因为Harpoxy的日志和其他Nginx之类略有不同，它是通过syslog协议讲日志发送出去的，
要想展现日志还要开启syslog进行收集。Rancher v1.2版本开始，
Rancher LB的功能已经解耦成独立的镜像rancher/lb-service-haproxy，如此日志收集的工作就可以做了。

### 开启访问日志
lb-service-haproxy的实现可以仔细阅读源码<https://github.com/rancher/lb-controller>，
简单的说，其内置了rsyslog来收集haproxy的日志，在容器内部查看配置文件**/etc/haproxy/rsyslogd.conf**，
可以知道其开启了udp 8514端口，不同的级别的日志发送到不同的文件中：
```
$ModLoad imudp.so
$UDPServerRun 8514
$template CustomFormat,"%timegenerated% %HOSTNAME% %syslogtag%%msg%\n"
$ActionFileDefaultTemplate CustomFormat

/* info */
if $programname contains 'haproxy' and $syslogseverity == 6 then (
    action(type="omfile" file="/var/log/haproxy/traffic")
)
if $programname contains 'haproxy' and $syslogseverity-text == 'err' then (
    action(type="omfile" file="/var/log/haproxy/errors")
)
/* notice */
if $programname contains 'haproxy' and $syslogseverity <= 5 then (
    action(type="omfile" file="/var/log/haproxy/events")
)

*.* /var/log/messages
```
关于syslog的Severity level可以参考这里：<https://en.wikipedia.org/wiki/Syslog>，
所以我们要查看的访问日志，应该就是/var/log/haproxy/traffic文件。
同时还添加了logrotate的配置进行日志分割和压缩，可查看容器内的**/etc/logrotate.d/haproxy**文件。

当前lb-service-haproxy版本默认是不开启日志收集的，需要自己手动开启，
我们都知道Haproxy开启日志需要在其global和defaults配置中添加log的配置。
Rancher LB是支持custom haproxy.cfg的，所以在Rancher中添加这两块配置就可以这样：  
![](https://ww2.sinaimg.cn/large/006tKfTcjw1fbgy5m62l3j30i80ecgn1.jpg)  
Rancher会自动合并这几条配置，最终在lb-service-haproxy容器内生成的配置就是这样：
```
global
    log 127.0.0.1:8514 local0
    log 127.0.0.1:8514 local1 notice
    chroot /var/lib/haproxy
    daemon
    group haproxy
    maxconn 4096
.....
.....
defaults
    log global
    option tcplog
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
.....
.....
```

那么查看之前分析的日志目录文件，可以看到响应的访问日志信息：  
![](https://ww1.sinaimg.cn/large/006tKfTcjw1fbgya5nb4vj30l309vn1r.jpg)

### 注意事项
日志虽然可以这样灵活的开启，但是使用时需要注意：

1. 日志文件放在容器内会不断增大，虽然有logrotate，但是扛不住日积月累。
临时测试用本地可以，但是长远看还是发送到外部syslog服务上比较好。
2. lb-service-haproxy的早期版本是默认就开启了访问日志，v0.4.5版本后取消了这个默认配置，
所以你就得使用我上面所说的方式，早期的lb-service-haproxy会积存很多访问日志，
所以尽早升级到v0.4.5以上的版本。

