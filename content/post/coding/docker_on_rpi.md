+++
images = [
]
tags = [
    "RancherOS",
    "Raspberry Pi"
]
date = "2016-11-07T17:36:37+08:00"
title = "树莓派上的Docker集群管理"
slug = "docker-on-rpi"
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""

+++
随着IOT市场的火热发展，Docker天然的轻量级以及帮助业务快速重构的特性，
将会在IOT领域迎来巨大发展潜力，甚至有可能会比它在云端的潜力更大。
本文将致力于构建一个利用RancherOS来管理运行在树莓派上的容器集群。
<!--more-->
### 引言
目前业界主流基本都是在x86架构上使用Docker，除了因为Intel在服务器领域的绝对领导地位之外，
x86 CPU的确在性能上有着卓越的表现。但是近些年来，随着云计算的迅猛发展，
引来了数据中心的大规模建设，慢慢地大家对数据中心PUE尤其是CPU功耗有了更高的要求。
ARM CPU虽然性能不如x86，但是在功耗上绝对有着无法比拟的优势，
同时我们知道并不是所有的服务都有高性能的CPU需要。很多厂商在都对ARM服务器投入了研发资源，
但是效果上目前来看并不是太好，ARM处理器在服务器领域并没有如在移动端那样被快速接受，
主要是因为市场接受度及服务器市场的性能要求所致。  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa26h172hoj20ib09dgmz.jpg)

但是在物联网（IOT）领域，ARM处理器却是霸主级的地位，毕竟在这个领域功耗胜过一切。
那么我们可以想象，未来会占大市场的IOT设备中，会出现各种尺寸各种架构，
内置操作系统也不统一，没有通用程序打包标准，几乎每种设备程序的开发框架均不同，
IOT设备中程序部署升级回滚等操作不够灵活，等等这样那样的问题。

### 分析与实践
这些问题，我们可以借鉴X86时代的经验，用Docker容器来解决它们。
Docker能降低IOT应用管理的负载度，但是在物理设备和Docker之间，
我们还需要一个轻量级的操作系统。这个OS需要是完全可以定制的，
可以针对不同设备需求，裁剪或增加对应的程序模块，更小体积更少进程意味着更低的功耗。

根据以上判断和需求，我经过了一番探索，最终选择了RancherOS。它本身的特性是：

1. 真正容器化的Linux操作系统极致精简，所有服务（包括系统服务）均运行在容器中，
  可以以容器方式对其进行任意定制
2. 内置了Docker Engine，无需在安装系统后再进行Docker安装
3. 完全开源<https://github.com/rancher/os>，我们可以进行各种深度定制
4. 最最重要的，支持ARM

在RancherOS的整体架构中，最底层毋庸置疑是Linux kernel，系统启动后的PID 1用system-docker代替，
由它来把udev、dhcp、console等系统服务启动，同时会启动user-docker，
用户运行的应用程序均跑在user-docker下。  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa26kta7rkj20l109s3zt.jpg)

我们以树莓派为例，将RancherOS部署在其之上。这里需要提示的是，RancherOS每个版本release之时，
都会放出树莓派的支持版本，
比如本次分享使用的[v0.7.0版本](https://github.com/rancher/os/releases/
download/v0.7.0/rancheros-raspberry-pi.zip)。通过dd命令将RancherOS写到树莓派的SD卡上，
通电点亮树莓派。  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa26m5gaz0j20aq0aawg4.jpg)

查看PID 1是system-docker：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa26mgfy51j20fn03ldh0.jpg)

通过system-docker ps 查看启动的系统服务：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa26mp3zphj20s603u0uw.jpg)

正常来说，我们都得设置一下docker registry mirror，这样方便pull镜像。
RancherOS的配置，都是通过ros config命令来配置，比如设置user-docker的mirror：
```
$ sudo ros config set rancher.docker.extra_args [--registry-mirror,https://xxxxxxx]
$ sudo system-docker restart docker # 重启user-docker
```
最终，可以看到：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa26nqlvixj20wf02ydi4.jpg)

RancherOS有一个我认为比较好的特性，就是支持很方便的对Docker Engine版本进行切换。
目前Docker迭代的速度并不慢，实际上很多程序不一定会兼容比较新的Engine，
Docker Engine版本的管理变得越来越重要。尤其是在测试环境中，
我们有时确实需要变换Docker Engine版本，来构建测试场景：
```
$ sudo ros engine list  #查看当前版本支持的engine有哪些
disabled docker-1.10.3
disabled docker-1.11.2
current  docker-1.12.1
$ sudo ros engine switch docker-1.11.2 #切换docker-1.11版本
```

此外，如果对docker engine有更特殊的需求，还可以定制自己的版本，然后让system-docker来加载它。
只需将编译好的docker engine放到scrach镜像中即可：  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa26p7gcphj20fx06fmy1.jpg)  
这部分可以参考：<https://github.com/rancher/os-engines>

另外，如果习惯了使用相应Linux发行版的命令行，
那么也可以加载对应的console镜像（当然如果考虑精简系统也可不必加载）：  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa26pvzryej20by03pdg5.jpg)  
此部分需要进行深度定制，可以参考：<https://github.com/rancher/os-images>  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa26qd06b7j20ej0bm410.jpg)

RancherOS更多酷炫的功能，可以访问官方的文档：<http://docs.rancher.com/os>

RancherOS介绍完毕后，我们可以在单机树莓派上做容器管理了，喜欢命令行的当然最好，
喜欢UI管理的，推荐两款可以在树莓派上运行的管理程序。
**portainer**<https://github.com/portainer/portainer>，
其有专门的arm镜像portainer/portainer:arm ，运行后：
`$ docker run --restart=always -d -p 9000:9000 --privileged 
    -v /var/run/docker.sock:/var/run/docker.sock 
    portainer/portainer:arm`  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa26s6xjj2j20oz0e2jtp.jpg)

更加简约的 ui-for-docker https://github.com/kevana/ui-for-docker ，运行如下：
`$ docker run --restart=always -d -p 9000:9000 
    -v /var/run/docker.sock:/var/run/docker.sock 
    hypriot/rpi-dockerui`  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa26sxv8ofj20ot0g0abm.jpg)

单机树莓派之后，我们就要考虑如何将多个树莓派组成Docker集群。
一提到Docker集群，我们就会考虑需要编排引擎的支持，
无非就是主流的Mesos、Kubernetes、Swarm，还有非主流的Cattle、Nomad之流。
那么在IOT场景下，我们最需要考虑的就是精简，所以我选择了新版的Swarm。
将RancherOS的Engine切换到1.12.3，然后构建Swarm集群。

简单得执行swarm init和join后，我们就得到了一个树莓派Docker集群：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa26tjxxlyj20fi01twey.jpg)

然后我们可以快速执行一个小demo：
```
#swarmkit demo
$ docker service create --replicas 1 -p 80 --name app armhf/httpd
$ docker service scale app=2
$ docker service ps app
```  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa26u6875lj20jk01y751.jpg)

更多ARM相关的Docker镜像，可以到这两个地方查找：

* <https://hub.docker.com/r/armhf>
* <https://hub.docker.com/u/aarch64>

RancherOS设计之初是为了构建一个运行Rancher的轻量级操作系统，
那么Rancher本身在ARM的支持上也在不断推进中，
相应的[PR](https://github.com/rancher/rancher/pull/4704)也有提交。
不过目前来看，对rancher-server的ARM化还是比较麻烦，对agent的节点支持ARM相对简单一些，
也就是说rancher-server仍然运行在x86架构上，而agent节点可以支持ARM和x86。

Kubernetes的ARM支持在社区中也有很多人在做，比如：
<https://github.com/luxas/kubernetes-on-arm>，来自社区的分享：  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa26w8ifrrj20i00h577r.jpg)

秀一下，我的“家庭树莓派数据中心”：  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa26wjj274j20c50chmzj.jpg)

---
最后，我要特别感谢RancherOS的开发者们，他们帮助我解决了很多问题；
另外还要特别感谢MBH树莓派社区的伙伴，提供了硬件设备，支持我的技术探索，
并提供了很多帮助。

