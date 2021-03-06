+++
images = [
]
tags = [
    "Rancher",
    "Networking",
]
categories = [
    "coding"
]
date = "2017-05-16T16:31:50+08:00"
title = "Rancher下实现Calico/Flat/Macvlan网络"
slug = "rancher-extend-network"
description = ""
menu = ""
banner = ""

+++
Rancher下的高性能网络方案
<!--more-->
### 引言
目前Rancher的开源版本中提供了IPSec和VXLAN两种网络方案：IPSec更多面向的是安全加密的场景，
但是网络性能就会差很多；Vxlan与IPSec同属Overlay模型，但是内核层面的支持和没有加密环节的损耗，
相对来说Vxlan的性能要远高于IPSec，在AWS上测试的Vxlan带宽性能是逼近于两台VM之间带宽，
当然由于AWS本身的网络质量就很好，用户在自己的环境测试时并不会有这么好的性能。
当面向私有数据中心部署时，Overlay的模型并不一定满足用户的需求，而且当前容器网络发展非常迅猛，
出现了很多独树一帜的网络插件，所以这里我选择了三种插件Calico、Flat、Macvlan对接Rancher，
希望可以为更多用户提供帮助。

### Rancher对接第三方网络插件
Rancher在v1.2版本重新架构后，实际上已经为第三方网络插件的支持留下了空间。Rancher中的network-manager
组件是管理网络插件的核心，其管理原理本文不赘述，后面会放一个PPT参考链接，有兴趣可以仔细品味。
但是当真正对接第三方插件时，还有一些小问题需要解决，这个问题就是多个ENV时，每个ENV都有独立的网络插件，
那么network-manager如何匹配自己当前ENV的CNI config文件，关于此问题，
我建了一个issue[#8535](https://github.com/rancher/rancher/issues/8535)，并提交了相关PR。
在Rancher v1.6.0+版本中，已经包含了这些修复。

此外，还需要注意的是，在Rancher中集成第三方网络插件需要满足以下三个条件，否则兼容性和体验会大打折扣：

1. 容器可以分配IP，并能互相联通，IP与MAC设定尽量与Rancher同步
2. 容器网络可以访问metadata服务，即容器内可以ping rancher-metadata
3. LB与metadata等组件兼容性

### Calico的支持
Calico本身的设计思路很独到，利用BGP技术学习路由，通过路由构建高性能的网络，但是Calico本身在用户端的落地阻力还是相当大的，
很多用户纯粹把Calico的支持当作一个Poc的要点，因为如果用户真的决定使用Calico网络，那么来自用户内部网管的阻力是相当大的。
BGP路由动态学习对于很多行业，这种网络几乎认为是没有管理的状态。

出于对Poc支持的考虑和一些付费用户的定制需求，大家如有兴趣可以参考一个demo版本的Rancher Calico的实现：

1. Issue描述[#rancher-8603](https://github.com/rancher/rancher/issues/8603)
2. Demo Catalog [#test-calico-catalog-item](https://github.com/niusmallnan/test-calico-catalog-item)，请使用test分支

### Flat的支持
Flat网络，中国用户喜欢称它为扁平网络，其目的是构建一种不带有任何路由或者overlay技术的网络，基本上容器的网络就是在一张大网中。
通常容器的Flat网络是在交换机中配置好的，主机和容器其实使用的是同一个网段，这时候需要规划好网络的设置，比如主机的ip范围和容器的ip范围不能冲突。

Flat网络的实现非常简单，原理上用户也很容易了解，在用户自身的网络部门推广起来几乎没有阻力，很多商业用户依赖这个网络。
所以Rancher已经开源了Flat网络的实现，用户可以在v1.6.11版本上使用Flat网络，在Catalog中就可以找到。

### Macvlan的支持
Macvlan网络更多是在Docker的CNM下被提及，其实CNI下也有Macvlan的实现，通常我们利用Macvlan可以隔离不同的容器子网。
但是实际上这个特性Rancher并不是十分需要，Rancher的设计中一个ENV下就是一张网，这样可以很方便用户使用。
如果不需要Macvlan的隔离特性，实际上这里其实更多就是利用Macvlan构建了一个类似Flat的网络，对于大部分Rancher用户来说Macvlan并没有很大的使用价值。

出于对Poc支持的考虑和一些付费用户的定制需求，大家如有兴趣可以参考一个demo版本的Rancher Macvlan的实现：

1. Issue描述[#8686](https://github.com/rancher/rancher/issues/8686)
2. Demo Catalog App [macvlan](https://github.com/niusmallnan/flatnet-catalog)

### 总结
更多内容可以参考我之前对外分享的PPT，请自行搭梯子，[Rancher network management](https://docs.google.com/presentation/d/1U4YZpsXnFg6l7YNIcaq0SJ5DsUlSr14mtWnxQ-rvFwQ/edit?usp=sharing).
