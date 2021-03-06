+++
images = [
]
date = "2016-10-08T16:09:24+08:00"
title = "如何hack一下rancher k8s"
slug = "rancher-k8s-hacking"
tags = [
    "Rancher",
    "Kubernetes",
]
categories = [
    "coding"
]
description = ""
menu = ""
banner = ""

+++
Rancher可以轻松实现Kubernetes的部署，尽管默认的部署已经完全可用，
但是如果我们想修改部署的K8s版本，这时应当如何应对？
<!--more-->
### 原理分析与执行  
在Rancher中，由于K8s是基于Cattle引擎来部署，所以在K8s在部署完成之后，
我们可以通过Link Graph来很清晰的看到整体的部署情况。  
![](http://ww1.sinaimg.cn/large/006tNc79jw1fa0ybyprq4j30mi07tdh0.jpg)

既然基于Cattle引擎部署，也就是说需要两个compose文件，
k8s引擎的compose文件放在<https://github.com/rancher/rancher-catalog/tree/master/templates>下面，
这里面有两个相关目录kubernetes与k8s，k8s是Rancher1.2开始使用的，
而kubernetes则是Rancher1.2之后开始使用的。  
![](http://ww1.sinaimg.cn/large/006tNc79jw1fa0ydzd1dgj30gt07qmyg.jpg)

为了我们可以自己hack一下rancher k8s的部署，我们可以在github上fork一下rancher-catalog，
同时还需要修改一下Rancher中默认的catalog的repo地址，
这个可以在<http://rancher-server/v1/settings>页面下，
寻找名为 catalog.url 的配置项，然后进入编辑修改。比如我这里将library库的地址换成了自己的：
<https://github.com/niusmallnan/rancher-catalog.git>  
![](http://ww3.sinaimg.cn/large/006tNc79jw1fa0yff7sbzj30h206imyy.jpg)

此时，我们就可以修改了，找一个比较实用的场景。我们都知道k8s的pod都会依赖一个基础镜像，
这个镜像默认的地址是被GFW挡在墙外了，一般我们会把kubelet的启动参数调整一下，
以便重新指定这个镜像地址，比如指定到国内的镜像源上。  
**--pod-infra-container-image=index.tenxcloud.com/google_containers/pause:2.0**。

如果我们要让rancher k8s部署时自动加上该参数，
可以直接修改私有rancher-catalog中的k8s compose文件。  
![](http://ww4.sinaimg.cn/large/006tNc79jw1fa0ygscal6j30gv0asjtp.jpg)

修改之后稍等片刻（主要是为了让rancher-server更新到新的catalog compose文件），
添加一个k8s env并在其中添加host，k8s引擎就开始自动部署，
部署完毕后，我们可以看到Kubernetes Stack的compose文件，
已经有了--pod-infra-container-image这个启动参数。  
![](http://ww4.sinaimg.cn/large/006tNc79jw1fa0yhg3alsj30gn09rtav.jpg)  
如此我们在添加pod时再也不用手动导入pod基础镜像了。

在compose file中，部署k8s的基础镜像是rancher/k8s，这个镜像的Dockerfile在rancher维护的k8s分支中，
如在rancher-k8s 1.2.4分支中可以看到：  
![](http://ww2.sinaimg.cn/large/006tNc79jw1fa0yi5rmplj30e007ogmr.jpg)

这样如果想对rancher-k8s发行版进行深度定制，就可以重新build相关镜像，通过rancher-compose来部署自己的发行版。

### 总结  
本文写于Rancher1.2行将发布之际，1.2版本是非常重大的更新，Rancher会支持部署原生的K8s版本，
同时CNI网络和Cloud Provider等都会以插件方式，用户可以自己定义，并且在UI上都会有很好的体现。
只要了解Rancher部署K8s的原理和过程，我们就可以定制非常适合自身使用的k8s，
通过Rancher来部署自定义的k8s，我们就可以很容易的扩展了k8s不擅长的UI、Catalog、
用户管理、审计日志维护等功能，这也是本文的目的。

