+++
title = "Rancher v1.2网络架构解读"
slug = "rancher12-networking"
tags = [
    "Rancher",
    "Rancher v1.2",
]
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""
images = [
]
date = "2016-12-01T21:18:36+08:00"

+++
Rancher v1.2系列文章，已独家授权Rancher Labs，转载请联系Rancher China相关人员。
<!--more-->
### 引言
在之前的Rancher版本上，用户经常诟病Rancher的网络只有IPsec，没有其他选择。
而容器社区的发展是十分迅猛的，各种容器网络插件风起云涌，欲在江湖中一争高下。
Rancher v1.2版本中与时俱进，对之前的网络实现进行了改造，支持了CNI标准，
除IPsec之外又实现了呼声比较高的VXLAN网络，同时增加了CNI插件管理机制，
让我们可以hacking接入其他第三方CNI插件。本文将和大家一起解读一下Rancher v1.2中网络的实现。

### Rancher-net CNI化
以最简单最快速方式部署Rancher并添加Host，以默认的IPsec网络部署一个简单的应用后，
进入应用容器内部看一看网络情况，对比一下之前的Rancher版本：  
![](http://ww2.sinaimg.cn/large/006tKfTcjw1fav9di74b7j30iv06tgoo.jpg)  
我们最直观的感受便是，网卡名从eth0到eth0@if8有了变化，原先网卡多IP的实现也去掉了，
变成了单纯的IPsec网络IP。这其实就引来了我们要探讨的内容，虽然网络实现还是IPsec，
但是rancher-net组件实际上是已经基于CNI标准了。最直接的证明就是看一下，
rancher-net镜像的Dockerfile：  
![](http://ww4.sinaimg.cn/large/006tKfTcjw1fav9e5komyj30ju06o40i.jpg)
熟悉CNI规范的伙伴都知道/opt/cni/bin目录是CNI的插件目录，bridge和loopback也是CNI的默认插件，
这里的rancher-bridge实际上和CNI原生的bridge没有太大差别，只是在幂等性健壮性上做了增强。
而在IPAM也就是IP地址管理上，Rancher实现了一个自己的rancher-cni-ipam，它的实现非常简单，
就是通过访问rancher-metadata来获取系统给容器分配的IP。
Rancher实际上Fork了CNI的代码并做了这些修改，<https://github.com/rancher/cni>。
这样看来实际上，rancher-net的IPsec和Vxlan网络其实就是基于CNI的bridge基础上实现的。

在解释rancher-net怎么和CNI融合之前，我们需要了解一下CNI bridge模式是怎么工作的。
举个例子，假设有两个容器nginx和mysql，每个容器都有自己的eth0，
由于每个容器都是在各自的namespace里面，所以互相之间是无法通信的，
这就需要在外部构建一个bridge来做二层转发，
容器内的eth0和外部连接在容器上的虚拟网卡构建成对的veth设备，这样容器之间就可以通信了。
其实无论是docker的bridge还是cni的bridge，这部分工作原理是差不多的，如图所示：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fav9ffzi1sj30ch08q750.jpg)

那么我们都知道CNI网络在创建时需要有一个配置，这个配置用来定义CNI网络模式，
读取哪个CNI插件。在这个场景下也就是cni bridge的信息，
这个信息rancher是通过rancher-compose传入metadata来控制的。
查看ipsec服务的rancher-compose.yml可以看到，type使用rancher-bridge，
ipam使用rancher-cni-ipam，bridge网桥则复用了docker0，
有了这个配置我们甚至可以随意定义ipsec网络的CIDR，如下图所示：  
![](http://ww2.sinaimg.cn/large/006tKfTcjw1fav9g15ofmj30c80anab3.jpg)

ipsec服务实际上有两个容器：一个是ipsec主容器，内部包含rancher-net服务和ipsec需要的charon服务；
另一个sidekick容器是cni-driver，它来控制cni bridge的构建。两端主机通过IPsec隧道网络通信时，
数据包到达物理网卡时，需要通过Host内的Iptables规则转发到ipsec容器内，
这个Iptables规则管理则是由network-manager组件来完成的，
<https://github.com/rancher/plugin-manager>。其原理如下图所示（以IPsec为例）：  
![](http://ww4.sinaimg.cn/large/006tKfTcjw1fav9gp2vkwj30hj06pacf.jpg)  
整体上看cni ipsec的实现比之前的ipsec精进了不少，而且也做了大量的解耦工作，
不单纯是走向社区的标准，之前大量的Iptables规则也有了很大的减少，性能上其实也有了很大提升。

### Rancher-net vxlan的实现
那么rancher-net的另外一个backend vxlan又是如何实现的呢？
我们需要创建一套VXLAN网络环境来一探究竟，默认的Cattle引擎网络是IPsec，
如果修改成VXLAN有很多种方式，可以参考我下面使用的方式。

首先，创建一个新的Environment Template，把Rancher IPsec禁用，同时开启Rancher VXLAN，如下图所示：  
![](http://ww2.sinaimg.cn/large/006tKfTcjw1fav9hpxm04j30ft09wmya.jpg)  
然后，我们创建一个新的ENV，并使用刚才创建的模版Cattle-VXLAN，创建完成后，
添加Host即可使用。如下图所示：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fav9i7um3fj30d7083js5.jpg)  
以分析IPsec网络实现方式来分析VXLAN，基本上会发现其原理大致相同。
同样是基于CNI bridge，使用rancher提供的rancher-cni-bridge、rancher-cni-ipam，
网络配置信息以metadata方式注入。区别就在于rancher-net容器内部，
rancher-net激活的是vxlan driver，它会生成一个vtep1042设备，并开启udp 4789端口，
这个设备基于udp 4789构建vxlan overlay的两端通信，对于本机的容器通过eth0走bridge通信，对于
其他Host的容器，则是通过路由规则转发到vtep1042设备上，再通过overlay到对端主机，
由对端主机的bridge转发到相应的容器上。整个过程如图所示：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fav9j4nhblj30e607z0um.jpg)  
vxlan实现的原理可更多参考：<https://www.kernel.org/doc/Documentation/networking/vxlan.txt>

### 总结
容器网络是容器云平台中很重要的一环，对于不通规模不通的安全要求会有不同的选型。
Rancher的默认网络改造成了CNI标准，同时也会支持其他第三方CNI插件，
结合Rancher独有的Environment Template功能，用户可以在一个大集群中的每个隔离环境内，
创建不同的网络模式，以满足各种业务场景需求，这种管理的灵活性是其他平台没有的。
