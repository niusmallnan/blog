+++
images = [
]
date = "2016-12-05T11:14:19+08:00"
title = "go-machine-service访问Rancher API的授权机制分析"
slug = "go-machine-service-access-api-key"
tags = [
    "Rancher",
]
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""

+++
一段分析调用机制的旅程。
<!--more-->
### 引言
深更半夜一位道上的朋友突然微信问我，Rancher Server内Cattle之外的服务如何调用Rancher API。
调用API本身没问题，但是Rancher API是需要授权访问的，
也就是说Rancher Server内其他服务如何授权访问API，这个机制是什么样的？
Rancher Server内有很多服务，如catalog、go-machine-service、websocket-proxy等等，
本文就以go-machine-service为例说明其授权机制。

### 以go-machine-service为例
首先我们要了解go-machine-service在做什么事，官方文档里很清楚的描述到，
它实现了physicalhost相关事件的handler，主要包括：

* physicalhost.create - Calls machine create ....
* physicalhost.activate - Runs docker run rancher/agent ...
* physicalhost.delete|purge - Calls machine delete ...

而若实现这些handler，需要和rancher events打交道，
我们认为rancher events也是rancher api的一部分，
所以访问rancher events也需要和API服务一样的鉴权信息。
go-machine-service在启动之时，很明确的读取了CATTLE相关变量：  
![](http://ww3.sinaimg.cn/large/006y8lVajw1fafqwxo6pyj30ir0ccdhx.jpg)

而且rancher还给go-machine-service专门创建了相应的API Key，查看DB可以得知：  
![](http://ww3.sinaimg.cn/large/006y8lVajw1fafqxbv1wbj30f90iwgnb.jpg)

进入rancher-server容器内部，查看go-machine-service进程的环境变量，
可以看到API Key的授权信息就在其中：  
![](http://ww3.sinaimg.cn/large/006y8lVajw1fafqxofpl1j30m805dmz0.jpg)

那么go-machine-service的环境变量是如何设置的呢？查看Cattle源码，找到MachineLancher，
它是启动go-machine-service服务的具体实现，我们可以看到设置环境变量这一步：  
![](http://ww1.sinaimg.cn/large/006y8lVajw1fafqy3ywfbj30gg03tt9w.jpg)

### 总结
Cattle本身就是一个框架，Rancher Server内的其他服务都是通过Cattle的Proxy机制来转发请求的，
授权机制则是由Cattle统一提供出来，其他服务自身无需单独实现授权机制。
如果我们想对Rancher Server扩展，再增加一些其他服务，利用本文所描述的机制，就会非常方便。


