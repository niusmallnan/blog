+++
banner = ""
date = "2017-03-09T15:45:41+08:00"
title = "Kubelet无法访问rancher-metadata问题分析"
slug = "analysis-of-kubelet-start-failure"
images = [
]
tags = [
    "Rancher",
    "Kubernetes",
]
categories = [
    "coding",
]
description = ""
menu = ""

+++
拯救一脸懵逼
<!--more-->
### 引言
Rancher能够支持Kubernetes，可以快速几乎无障碍的拉起一套K8s环境，这对刚入门K8s的小白来说简直是一大利器。
当然由于系统特性五花八门，系统内置软件也相互影响，所以有时候伙伴们会碰到比较难缠的问题。
本文就分析一下关于kubelet无法访问rancher-metadata问题，当然这个问题并不是必现，
这里要感谢一位Rancher社区网友的帮助，他的环境中发现了这个问题，并慷慨得将访问权限共享给了我。

### 问题现象
使用Rancher部署K8s后，发现一切服务状态均正常，这时候打开K8s dashboard却无法访问，
细心得查看会发现，dashboard服务并没有部署起来，这时下意识的行为是查看kubelet的日志，
此时会发现一个异常：  
![](https://ww1.sinaimg.cn/large/006tNbRwly1fdgnj5h1bjj30g503omxw.jpg)  
你会发现kubelet容器内部一直无法访问rancher-metadata，查看rancher-k8s-package源码，
kubelet服务启动之前需要通过访问rancher-metadata做一些初始化动作，由于访问不了，
便一直处于sleep状态，也就是出现了上面提到的那些异常日志的现象：  
![](https://ww2.sinaimg.cn/large/006tNbRwly1fdgnn1tbzzj30k2071q41.jpg)

同样，在github上也能看到类似的issue：<https://github.com/rancher/rancher/issues/7160>

### 排查分析
进入kubelet容器一探究竟，分别用ping和dig测试对rancher-metadata访问情况如下：  
![](https://ww1.sinaimg.cn/large/006tNbRwly1fdgnq66xs1j30ey0abmyh.jpg)  
dig明显可以解析，但是ping无法解析，因此基本排除了容器内dns nameserver或者网络链路情况的问题。

既然dig没有问题，ping有问题，那么我们就直接采取使用strace（`strace ping rancher-metadata -c 1`）来调试，
这样可以打印系统内部调用的情况，可以更深层次找到问题根源：  
![](https://ww1.sinaimg.cn/large/006tNbRwly1fdgnsufow0j30k7090tb2.jpg)  

之前提到这个问题并不是必现的，所以我们找一个正常的环境，同样用strace调试，如下：  
![](https://ww2.sinaimg.cn/large/006tNbRwly1fdgnu63kl2j30m0077dhj.jpg)

对这两张图，其实已经能够很明显的看出区别，有问题的kubelet在解析rancher-metadata之前，
向nscd请求的解析结果，nscd返回了unkown host，所以就没有进行dns解析。
而正常的kubelet节点并没有找到nscd.socket，而后直接请求dns进行解析rancher-metadata地址。

经过以上的分析，基本上断定问题出在nscd上，那么为什么同样版本的rancher-k8s，
一个有nscd socket，而另一个却没有，仔细看一下kubelet的compose定义：  
![](https://ww4.sinaimg.cn/large/006tNbRwly1fdgny3wpihj30c5087wf5.jpg)  
kubelet启动时候映射了主机目录/var/run，那么基本可以得知nscd来自于系统。
检查一下有问题的kubelet节点的系统，果然会发现安装了nscd服务（服务名为unscd）。

用比较暴力的方案证明一下分析过程，直接删除nscd socket文件，这时候你会发现kubelet服务正常启动了，
rancher-metadata也可以访问了。

回过头来思考一下原理，为什么ping/curl这种会先去nscd中寻找解析结果呢，而dig/nslookup则不受影响。
ping/curl这种在解析地址前都会先读取/etc/nsswitch.conf，这是由于其底层均引用了glibc，
由nsswitch调度，最终指引ping/curl先去找nscd服务。nscd服务是一个name services cache服务，
很多解析结果他会缓存，而我们知道这个nscd是运行在Host上的，Host上是不能直接访问rancher-metadata这个服务名，
所以kubelet容器中就无法访问rancher-metadata。

### 其他解决方案
其实我们也未必要如此暴力删除nscd，nscd也有一些配置，我们可以修改一下以避免这种情况，
可以disable hosts cache，这样nscd中便不会有相应内容的缓存，所以解析rancher-metadata并不会出现unknown host，
而是继续向dns nameserver申请解析地址，这样也不会有问题。  
![](https://ww4.sinaimg.cn/large/006tNbRwly1fdgo48mxf8j30ej06lab0.jpg)

### 总结
遇到问题不能慌，关键是要沉得住气，很多看似非常复杂的问题，其实往往都是一个小配置引发的血案。
