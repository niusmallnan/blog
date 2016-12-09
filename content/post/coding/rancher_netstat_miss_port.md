+++
title = "为什么Rancher v1.2中netstat看不到开放端口"
slug = "rancher-netstat-miss-port"
banner = ""
images = [
]
tags = [
]
categories = [
]
description = ""
menu = ""
date = "2016-12-09T15:07:14+08:00"

+++
拯救一脸懵逼。
<!--more-->
### 引言
随着Rancher v1.2的发布，越来越多的伙伴参与到新特性的尝鲜中。有时不免会碰到一些问题，
比如网络不通之类的，这时通常我们都会下意识的使用netstat命令查看端口是否正确开启，
可是出现的结果却让伙伴们一脸疑问。比如我们使用了ipsec网络，
netstat命令却看不到UDP 500和4500端口的开放，这是为什么呢？本文将释疑这个问题。

### 问题现象
启动之前的Rancher v1.1版本并对比v1.2版本，可以看到如下现象，
ipsec网络正常，但在新版中netstat却看不到端口开放：  
![](http://ww2.sinaimg.cn/large/006tKfTcjw1fakk2e84hcj30d509z76g.jpg)  
用LB暴露一个服务访问，UI上明明显示了端口，但是在netstat中却看不到：  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1fakk3aasx6j30j7075my6.jpg)  
这些现象是为什么？

### 原生docker和k8s可以看到端口
原先的docker，一般run一个容器加入-p参数，就可以看到端口。
这其实是由当前tcp/ip栈内的docker-proxy开放的，如图：  
![](http://ww2.sinaimg.cn/large/006tKfTcjw1fakk45w280j30ju06t0v2.jpg)  
k8s是可以通过netstat看到端口，举个例子，创建一个service暴露NodePort，如下图：  
![](http://ww4.sinaimg.cn/large/006tKfTcjw1fakk4l1qplj308z069wer.jpg)  
而通过netstat命令却能看到对应的端口，这是因为这个端口是由kube-proxy开放的，
就是说在这个tcp/ip栈内有应用程序bind了这个端口：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fakk4z18gzj30j605jtah.jpg)

### 本质原因
端口的本质其实是应用程序，就是说需要有process bind port，
而netstat要能显示出开放端口，则必须在当前的namespace里有应用程序监听了端口才行。
Rancher中因为是使用CNI模型，虽然也是使用docker0，但是这个docker0是CNI Bridge，
并不和docker-proxy有协作，所以端口并没有通过docker-proxy暴露，
只是通过iptables dnat转发到其他的namespace上。在当前的namespace里使用netstat看不到开放的端口，
iptables dnat规则如下：  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1fakk5qzfr6j30jg03idh7.jpg)

### 总结
有时候惯性思维让我们忽略了事物的本质，让人匪夷所思的问题往往是很简单的道理。
