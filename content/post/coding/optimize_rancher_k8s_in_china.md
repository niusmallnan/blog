+++
banner = ""
images = [
]
tags = [
    "Rancher",
]
categories = [
    "coding",
]
description = ""
menu = ""
date = "2017-01-19T17:32:56+08:00"
title = "优化Rancher k8s中国区的使用体验"
slug = "optimize-rancher-k8s-in-china"

+++
拯救一脸懵逼
<!--more-->
### 引言
Kubernetes（以下简称K8s）是Rancher平台重点支持的一个编排引擎，Rancher K8s具有部署灵活使用方便的特点，
而且Rancher基本是同步更新支持K8s的新版本新组件，用户也可以选择部署指定的K8s版本。
但是这一切的便利，身在中国的我们这些贱民无法深刻体验，万恶的GFW把很多部署的依赖挡在之外，
而服务全球开发者的Rancher平台亦不可能为中国用户单独定制，所以我只好自己动手丰衣足食。

### 部署要点
部署之前的操作系统选型上，相对来说我比较推荐ubuntu+docker的组合，
毕竟这个组合在国外使用的用户比较多，相对来说bug fix的速度也是比较快的，
如果你是一个docker重度用户，应该深知docker本身的bug并不少。

如果是部署一个新的Rancher环境，我推荐用下面的脚本来启动，通过设置DEFAULT_CATTLE_CATALOG_URL，
这样可以直接指定我定制过的Rancher K8s：
```
docker run -d --restart=unless-stopped \
     -e DEFAULT_CATTLE_CATALOG_URL='{"catalogs":{"community":{"url":"https://github.com/rancher/community-catalog.git","branch":"master"},"library":{"url":"https://github.com/niusmallnan/rancher-catalog.git","branch":"k8s-cn"}}}' \
     --name rancher-server \
     -p 8082:8080 rancher/server:stable
```

当然如果是已经部署的Rancher环境，那就需要在Rancher UI上，做一下简单的修改，
Disable已有的library catalog repo，指向我定制过的即可，注意branch的设置，网络状况不好的需要耐心等待重新拉取repo内容： 
![](https://ww3.sinaimg.cn/large/006y8lValy1fbw2toyl38j30s30a50u6.jpg)

在部署agent节点前，如果是一个干净的环境最好，但是如果是曾经做过agent节点，
尤其是之前部署过rancher k8s的，我强烈建议你执行一次大扫除，否则会出现各种意想不到的状况，
大扫除的脚本可以参考执行我的这个，具体都做了什么事可自行阅读：
```
#!/bin/bash

docker rm -f $(docker ps -qa)

rm -rf /var/etcd/

for m in $(tac /proc/mounts | awk '{print $2}' | grep /var/lib/kubelet); do
    umount $m || true
done
rm -rf /var/lib/kubelet/

for m in $(tac /proc/mounts | awk '{print $2}' | grep /var/lib/rancher); do
    umount $m || true
done
rm -rf /var/lib/rancher/

rm -rf /run/kubernetes/

docker volume rm $(docker volume ls -q)

docker ps -a
docker volume ls
```

### 一切OpenSource
如果你对我在其中的改动颇有疑虑，亦大可放心。我主要是改动两个地方：
fork了rancher-catalog建立了k8s-cn的分支，只要将Rancher的library catalog repo指向我的工程分支即可；
fork了kubernetes-package，每次Rancher K8s发布新版本，
我都会基于该版本建立一个CN分支（如：v1.5.1-rancher1-7-cn），
一切对于中国区的优化修改都会在这个分支上。最终我会更新出中国区的使用镜像，并push到镜像仓库上，
目前使用的是阿里云的镜像仓库（招牌比较大短时间内不会倒。。。）。

参考链接：

1. <https://github.com/niusmallnan/rancher-catalog>
2. <https://github.com/niusmallnan/kubernetes-package>

### 后续支持计划
截止到本文写作之时，我刚开始支持rancher-k8s v1.5.1-rancher1-7版本，并在Rancher v1.3.1版本上做了测试。
后续Rancher官方发布新版本，我会进行同步更新，并做一些简单的测试。
后续考虑加入离线安装，可以指定本地镜像仓库，依赖镜像一键导入等方便的功能。
当然如果在使用中发现各种疑难杂症，可以发邮件给我niusmallnan@gmail.com。

