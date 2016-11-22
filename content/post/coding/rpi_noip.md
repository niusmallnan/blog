+++
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""
date = "2016-11-22T13:59:30+08:00"
title = "基于容器实现树莓派的动态域名绑定"
slug = "rpi-noip-with-docker"
images = [
]
tags = [
    "Raspberry Pi",
    "Docker",
]

+++

<!--more-->
### 引言
我们在家里使用树莓派时，如果在上面搭建了一些服务，有时会期待能在外网可以访问。
家庭宽带会给我们分配一个外网IP，可以通过这个IP在公网上访问树莓派上的服务。
但是运营商提供的这个IP是非静态的，很可能在凌晨的时候会被重新分配，
而且IP也不是很好记，所以通常我们都希望能有一个域名可以直接访问。
动态域名解析是个老话题，国内最早花生壳就做过，具体原理不必多说。

### 我们的需求清单如下：

1. 一个动态域名解析的软件，最好是一定程度Free。
2. 域名解析的client端一定要是开源的（谁也不想被当肉鸡...）。
3. 部署控制要非常简单。

### 最终选型如下：

1. 使用国外[noip](https://www.noip.com/)的服务，免费账户可以绑定三个免费域名，
每个域名30天有效，失效后需要手动添加回来。这种程度对于我这种玩家已经完全足够了。
2. noip的client端支持各种平台，支持[Linux](https://www.noip.com/download?page=linux)，同时是开源。
3. 为了让部署变得更加简洁，决定使用Docker容器来部署。

### 执行过程

首先，需要到[noip](https://www.noip.com/)上注册账户，并填写自己的域名，注册过程请自行体验，比如：
![](http://ww2.sinaimg.cn/large/006tNc79jw1fa0utbt10oj30ja04maap.jpg)

然后，我们就需要在下载客户端，并在树莓派上进行编译，生成适合ARM运行的版本，编译只要执行make即可：
![](http://ww2.sinaimg.cn/large/006tNc79jw1fa0uxw14ulj30fk0a4wgz.jpg)

初次执行需要先在client端注册账号信息，其实就是生成一下相关配置文件，再次执行就可以运行起来了。

这么看来，对于很多用户来说还是太复杂，所以我们决定使用容器来简化这个过程。
可以把上面繁琐操作，全部放在容器中，为了精简程序的编译环境，
我们使用alpine-linux来进行编译，容器的Dockerfile如下：
![](http://ww4.sinaimg.cn/large/006tNc79jw1fa0uz7rv7oj30i304pmyk.jpg)

接着，我们需要对树莓派的系统进行容器化，以便我们可以运行noip容器程序，此时可以有三种选择：

1. RancherOS
2. HypriotOS
3. Rasbian 基础上安装Docker

前两个都是内置Docker Engine的，用起来比较方便，Rasbian需要自行安装Docker。我选择的OS是RancherOS。

### 最简方式部署noip，只需三步

在RancherOS上修改registry mirror后，可以加速镜像下载，然后拉取前面Dockerfile编译的镜像：

`$ docker pull hypriot/rpi-noip`

注册noip client，按照提示添加用户名密码等信息：

`$ docker run -ti -v noip:/usr/local/etc/ hypriot/rpi-noip noip2 -C`

运行noip clent：

`$ docker run -v noip:/usr/local/etc/ --restart=always hypriot/rpi-noip`

登录No-IP的[控制台](https://my.noip.com/#!/dynamic-dns)可以看到 dns绑定情况：
![](http://ww2.sinaimg.cn/large/006tNc79jw1fa0v2h5umhj30km05rjs0.jpg)

在本地使用dig确认一下解析情况：
![](http://ww2.sinaimg.cn/large/006tNc79jw1fa0v2po6wfj30gy08yjt4.jpg)

以上所有源码可以参考：<https://github.com/hypriot/rpi-noip>。
这样，我们就在树莓派上非常简单的实现了动态域名绑定。



