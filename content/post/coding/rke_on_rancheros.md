+++
menu = ""
date = "2018-01-21T16:23:54+08:00"
slug = "rke-on-rancheros"
title = "RancherOS上运行RKE"
banner = ""
images = [
]
tags = [
    "RancherOS",
    "RKE",
]
categories = [
    "coding",
]
description = ""

+++
RancherOS上运行RKE的注意事项
<!--more-->
### 注意事项
RancherOS是一个非常精简的操作系统，功能上肯定不如Ubuntu/CentOS之流全面，所以在其上面使用RKE部署Kubernetes时候需要注意一些问题。

#### 切换到合适的docker engine上
一般来说K8s并不会适配所有docker engine，但是RancherOS提供docker engine自由切换的功能，可以先列出RancherOS的支持的docker engine版本:

```
[root@ip-172-31-5-104 rancher]# ros engine list
disabled docker-1.12.6
disabled docker-1.13.1
disabled docker-17.03.1-ce
current  docker-17.03.2-ce
disabled docker-17.04.0-ce
disabled docker-17.05.0-ce
disabled docker-17.06.1-ce
disabled docker-17.06.2-ce
disabled docker-17.09.1-ce
```

根据RKE的文档，选择合适的docker engine，切换docker engine很简单，使用`ros engine switch xxx`即可。

#### 持久化相关目录
RancherOS的默认console中只有部分目录是持久化的，这意味着你重启主机后，非持久化目录的数据会自动清理。
K8s会有一些数据需要持久化的磁盘上，所以针对K8s这些持久化目录，我们需要提前给RancherOS配置额外的持久化目录。

RancherOS中目录的定义，请参考：http://rancher.com/docs/os/v1.1/en/system-services/system-docker-volumes/

RancherOS默认包括三个持久化目录(user-volumes)，`/home/` `/opt/` `/var/lib/kubelet`，如果要添加其他目录，可以使用下面的方式：

```
# 增加了 /etc/kubernetes 和 /etc/cni
ros config set rancher.services.user-volumes.volumes  [/home:/home,/opt:/opt,/var/lib/kubelet:/var/lib/kubelet,/etc/kubernetes:/etc/kubernetes,/etc/cni:/etc/cni]

system-docker rm all-volumes

reboot
```

具体那些目录需要持久化，还要看自己的部署需求，相关目录请参考：https://github.com/rancher/rke#cluster-remove

