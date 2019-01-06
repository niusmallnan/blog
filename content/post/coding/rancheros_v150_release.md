+++
images = [
]
description = ""
menu = ""
banner = ""
date = "2019-01-06T17:00:46+08:00"
title = "RancherOS v1.5.0发布"
slug = "rancheros-v150-release"
tags = [
    "RancherOS",
]
categories = [
    "coding",
]

+++
一言不合就发新版本
<!--more-->

年关将至，寒意习习，落叶萧萧下，阳光日日稀。RancherOS团队历时两个来月的开发，正式发布RancherOS v1.5.0版本。
在此期间同为Container Linux阵营的CoreOS已经从红帽再入IBM，潮流之变，业界态势，让我们无不更加努力去争得一席之地。
无论是商业用户的积累，还是业界变化带来的社区用户增长，都在催促我们不断革新，应该说1.5.0版本是用户的需求推着我们走出来的。

### 重大特性更新  
本版本的新特征众多，无法一次性全部说明，以下只表述一些用户关注度比较高的特性。个别特性详细说明，我们会不断推出文章一一展开。

#### 启动性能提升  
一直以来RancherOS的initrd一直采用xz格式压缩，随着RancherOS的体积不断增大，xz压缩越来越影响系统启动速度。虽然xz格式能够带来比较小的initrd和ISO，
但是我们也需要兼顾启动速度。v1.5.0版本的initrd已经采用了gzip格式，文件体积有所增大，但是启动速度有了质的飞跃。
同时我们也优化了system-docker的镜像加载和cloud-init的启动，对启动速度进行了深度优化。

#### LUKS磁盘加密支持  
支持LUKS，允许用户对跟磁盘分区进行加密，在一些特殊场景下增强了RancherOS的安全性。运行效果参考下图：

![](https://user-images.githubusercontent.com/4208881/47795869-ee87a780-dd2b-11e8-91be-4ac8f685d8fa.gif)

#### WiFi和4G支持  
Intel正在micro PC领域不断发力，RancherOS被纳入其生态体系，我们支持了WiFi和4G网络，用户可以通过简单的cloud-config配置就可以开启，
带来了十分简洁的用户体验，这部分我们会在后续其他文章中详细介绍。

#### Hyper-V支持  
很多社区用户一直希望能在Hyper-V使用RancherOS，先前我们一直提供给用户一些custom build的方式来实现它，现在我们正式支持了它，并会持续维护。
无论是docker-machine方式还是boot from ISO方式均可以支持。

下一个版本我们也会带来RancherOS的Azure Cloud支持。

#### 多docker engine支持  
这是一个很有趣的特性，目前RancherOS中默认拥有一个user docker。在v1.5.0中，用户可以用过ROS CLI来创建多个user docker engine，
并且每个docker拥有独立的ROOTFS和网络栈，并且可以在console很容易的切换使用任意一个docker。

当然我们并不推荐您在生产中使用，我们的某个商业客户把这个特性应用在其CI环境中，极大的提升了资源的利用率，减少了物理机器数量的开销。

#### 改善VMware的支持  
RancherOS的广大用户中Vmware是占有很大的用户群，之前我们的版本中只针对docker-machine方式做了一些改善，但是很多用户还希望使用boot from ISO方式和VMDK方式，
我们相关的镜像也做了支持，用户可以饿直接下载使用它：

- [VMDK] https://releases.rancher.com/os/v1.5.0/vmware/rancheros.vmdk
- [Docker Machine] https://releases.rancher.com/os/v1.5.0/rancheros-vmware.iso
- [Boot From ISO] https://releases.rancher.com/os/v1.5.0/vmware/rancheros.iso

#### ARM的支持  
由于Rancher和ARM已经开始了战略合作，我们会在一起做很多有趣的事。RancherOS的ARM支持也是其中的一部分，原先我们只是对RPi做了支持，
现在我们提供ARM版本的initrd和vmlinuz，用户可以用它们使用iPXE方式启动：

- https://releases.rancher.com/os/v1.5.0/arm64/initrd
- https://releases.rancher.com/os/v1.5.0/arm64/vmlinuz

我们依然只会对ARM64支持，且v1.5.0的ARM支持只是实验性质的，并不推荐应用在生产中。
我们会和ARM进行合作进行更广泛的测试，后续的版本将会是更稳定的。

### 更加友好的自定义  
社区中越来越多的发烧友并不局限使用我们的正式发布版本，他们会根据自己的需求修改RancherOS，构建自己的RancherOS。
我们提供了一些友好的编译选项，用户可以自定义自己的RancherOS。

#### 更改默认docker engine  
RancherOS的每个版本都会有自己设定的默认docker engine，而在用户的场景下，可能需要一个内部认可的docker engine，且希望它是RancherOS默认的版本。
那么用户可以在构建时候指定docker engine版本，来构建自己的RancherOS，以docker 17.03.2为例：

```
USER_DOCKER_VERSION=17.03.2 make release
```

#### 更改默认console  
RancherOS支持很多console，比如ubuntu、alpine、centos等，由于我们的default console基于busybox，有些用户并不喜欢它，且不希望每次都去切换console。
那么用户可以使用这种方式构建一个默认console是自己喜欢的版本，以alpine console为例：

```
$ OS_CONSOLE=alpine make release
```

### 其他  
AWS相关镜像已经上传到各个region中，可以直接搜索查找并使用，包括AWS中国区。其他主要镜像列表参考：https://github.com/rancher/os/blob/v1.5.x/README.md#release

更多新特性和Bug Fix请参考v1.5.0的[Release Notes](https://github.com/rancher/os/releases/tag/v1.5.0)

文档说明：https://rancher.com/docs/os/v1.x/en/

最后，RancherOS还是一个小众的开源项目，我们专注Docker在Linux上的精简体验，如果喜欢RancherOS，请在Github上给我们一个star，鼓励我们继续前行。
