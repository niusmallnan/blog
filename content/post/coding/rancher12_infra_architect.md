+++
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""
date = "2016-11-28T20:53:44+08:00"
title = "Rancher v1.2基础设施引擎整体架构"
slug = "rancher12-infra-architect"
images = [
]
tags = [
    "Rancher",
    "Rancher v1.2"
]

+++
Rancher v1.2系列文章，已独家授权Rancher Labs，转载请联系Rancher China相关人员。
<!--more-->
### 引言
Rancher v1.2可以说是一个里程碑版本，发布时间虽屡次跳票让大家心有不爽，
但是只要体会其新版功能，会发现这个等待绝对是值得的。从架构角度看，
用两个字来概括就是“解耦”，基础设施引擎的分离，agent节点的服务粒度更细；
从产品角度看，给了用户更多定制的空间，Rancher依然秉持着全部OpenSource的理念；
在开发语言上，之前遗留的通过shell脚本方式的粗糙实现也都基于Golang重写，
解耦的新服务也几乎使用Golang开发，agent节点全线基于Golang这也为后续便利地支持ARM埋下伏笔。
在市场选择上，Rancher依然在kubernetes下面投入了大量精力，引入了万众期待的CNI plugin管理机制，
坚持要做最好用的Kubernetes发行版。本文就带着大家从架构角度总览Rancher v1.2版本的特性。

### 总览
在v1.2版本的整体架构图（如下图所示）上，Cattle引擎向下深入演化成了基础设施引擎，
这一点上在v1.1时代也早有体现。Cattle更多得作为基础设施的管理工具，
可以用它来部署其他服务和编排引擎，当然它本身编排能力还是可以使用的，
习惯了stack-service的朋友仍然可以继续使用它，同时rancher scheduler的引入也大大增强了其调度能力。
Rancher仍然支持Kubernetes、Mesos、Swarm三大编排引擎，Kubernetes可以支持到较新的v1.4.6版本，
由于所有的部署过程的代码都是开放的，用户依然可以自己定制部署版本。值得一提的是，
Rancher支持了新版的Swarm Mode也就是Swarmkit引擎，这也意味着Rancher可以在Docker1.12上部署，
不小心装错Docker版本的朋友这回可以放心了。  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1fav8rinilzj30hj09p0ur.jpg)  
在存储方面，Rancher引入了Kubernetes社区的flexvol来做存储插件的管理，
同时也支持Docker原生的volume plugin机制，并实现了对AWS的EFS&EBS以及标准NFS的支持，
先前的Convoy应该会被抛弃，Rancher最终还是选择参与社区标准。在网络方面，
除了CNI插件机制的引入，用户还可以使用rancher-net组件提供的vxlan网络替代先前的ipsec网络。
在可定制性方面，还体现在Rancher提供了用户可以自定义rancher-lb的机制，
如果特殊场景下默认的Haproxy不是很给力时，用户可以自定义使用nginx、openresty或者traefik等等。
下面便做一下详细分解。

### 基础设施引擎
初次安装v1.2版本，会发现多了Infrastructure（如下图所示）的明显标识，
默认的Cattle引擎需要安装healthcheck、ipsec、network-services、scheduler等服务。
这个是有rancher-catalog来定义的，<https://github.com/rancher/rancher-catalog>，
新分离出来了infra-templates和project-templates：infra-templates就是Rancher定义的各种基础设施服务，
包括基础服务和编排引擎；project-templates对应的是Env初始化时默认安装的服务，
它可以针对不同的编排引擎进行配置。  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1fav8t4d321j30dl0b275b.jpg)  
以Cattle引擎为例，可以在project-templates的Cattle目录中找到相应的配置文件，
当ENV创建初始化时会创建这里面定义的服务，这样一个机制就可以让我们可以做更深入的定制，
让ENV初始化时创建我们需要的服务：  
```
name: Cattle
description: Default Cattle template
stacks:
- name: network-services
templateId: 'library:infra*network-services'
- name: ipsec
templateId: 'library:infra*ipsec'
- name: scheduler
templateId: 'library:infra*scheduler'
- name: healthcheck
templateId: 'library:infra*healthcheck'
```
在Cattle引擎调度方面，Rancher实现了rancher-scheduler，<https://github.com/rancher/scheduler>。
它实现了允许用户按计算资源调度，目前支持memory、cpu的Reservation。其实现原理是，
内部有一个resource watcher，通过监听rancher metadata的获取Host的使用资源数据变化，
进而得到ENV内所有Host资源汇总信息。与此同时，
监听rancher events的scheduler.prioritize、scheduler.reserve、scheduler.release等各种事件，
通过排序过滤可用主机后发送回执信息，Rancher Server就有了可以选择的Host列表。如下图所示：  
![](http://ww2.sinaimg.cn/large/006tKfTcjw1fav8vafyqaj30h005xwfk.jpg)  
需要注意的是，rancher-scheduler并没有和rancher-server部署在一起，
而是在你添加Host时候部署在agent节点上，当然rancher-scheduler在一个ENV内只会部署一个。

### Agent节点服务解耦
agent节点一个最大的变化就是，agent-instance容器没有了，它被拆分成多个容器服务，
包括rancher-metadata、network-manager、rancher-net、rancher-dns、healthcheck等。  
![](http://ww2.sinaimg.cn/large/006tKfTcjw1fav8w3brm9j30ea06labt.jpg)

metadata服务是老朋友了，它在每个agent节点上保存了host、stack、service、container等的信息，
可以非常方便的在本地调用取得，<https://github.com/rancher/rancher-metadata>，
在新的体系中它扮演了重要角色，几乎所有agent节点上的服务均依赖它。
在先前的体系中，metadata的answer file的更新是通过事件驱动shell脚本来执行下载，
比较简单粗暴。v1.2开始使用监听rancher events方式来reload metadata的answer file，
但是answer file还需要到rancher server端下载，总的来说效率还是有一定提升的。

dns在新的体系中仍然承担着服务发现的功能，<https://github.com/rancher/rancher-dns>。
除了拆分成单独容器之外，它也在效率上做了改进，它与rancher-metadata容器共享网络，
以metadata的结果生成dns的answer file。与之前的架构相比，
省去了dns answer file下载的过程。需要注意的是，rancher-dns的TTL默认是600秒，
如果出于各种原因觉得dns作为服务发现不是很可靠，那么可以使用etc-host-updater和rancher-metadata的组合，
[etc-host-updater](https://github.com/rancher/etc-host-updater)
会根据metadata数据动态生成hosts文件并写入容器内，这样通过服务名访问时，
其实已经在本地转换成了IP，无需经过dns，如下图所示：  
![](http://ww4.sinaimg.cn/large/006tKfTcjw1fav8z58gm6j30bc04mmxg.jpg)

rancher-net作出了比较重大的革新，<https://github.com/rancher/rancher-net>，
除了继续支持原有的ipsec外，还支持了vxlan。这个支持是原生支持，
只要内核有vxlan的支持模块就可以。vxlan并不是Cattle的默认网络，
使用时可以在infra-catalog中重新选择它来部署，其实现以及部署方式后续会在专门的文章中进行探讨：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fav8zxs951j30ds0al753.jpg)

network-manager的引入为Rancher v1.2带来了一个重要特性就是CNI插件管理，
在之前的版本中很多用户都提到rancher-net本身的网络无法满足业务需求。
容器网络之争，无非就是CNM与CNI，Rancher选择站队CNI，这也是为了更好与Kubernetes融合。
而CNI的插件很多种，Calico、Weave等之流，每个插件的部署方式都不一样。
Rancher为了简化管理提出了network-manager，<https://github.com/rancher/plugin-manager>，
它可以做到兼容主流的CNI插件，它实际上定义了一个部署框架，让CNI插件在框架内部署。
network-manager是以容器方式部署，由于每种插件在初始化时可能需要暴露端口或加入一些NAT规则，
所以network-manager能够动态设置不同插件的初始化规则，
它的做法是以metadata作为 host port和host nat规则的数据源，
然后获取数据后生成相应的Iptables规则加入的Host中。而对于真正的CNI插件，
需要在network-manager容器内/opt/cni目录下部署对应cni插件的执行程序（calico/weave），
/etc/cni目录下部署cni插件的配置，这两个目录映射了docker卷rancher-cni-driver，
也就是Host上的/var/lib/docker/volumes/rancher-cni-driver目录下。

关于healthcheck，先前是通过agent-instance镜像实现，里面内置了Haproxy，
事件驱动shell脚本来下载healchcheck配置并reload。新的架构中，
Rancher实现了单独的healthcheck，<https://github.com/rancher/healthcheck>，
采用Golang微服务的方式，数据源是metadata。
当然healthcheck的最终检查仍然是通过与Haproxy sock通信来查看相应member的健康状态（原理如下图），
healthcheck的实现主要是为了将其从agent-instance中解耦出来。  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1fav928q6bwj30hu06mad4.jpg)

### 总结
Rancher v1.2的新特性还是非常多的，基础设施引擎的变化时一切特性的基础，
这篇文章算是开篇之作。后续会持续为大家带来，Kubernetes、Swarmkit的支持，自定义rancher-lb，
vxlan的支持，各种CNI插件的集成，以及各种存储接入的实践操作指南等等。
