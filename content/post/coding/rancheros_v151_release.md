+++
title = "Runc CVE-2019-5736以及RancherOS v1.5.1发布"
description = ""
menu = ""
banner = ""
images = [
]
date = "2019-02-12T14:24:44+08:00"
slug = "rancheros-v151-release"
tags = [
    "RancherOS",
]
categories = [
    "coding",
]

+++
一言不合就发新版本
<!--more-->

runc是一个根据OCI(Open Container Initiative)标准创建并运行容器的CLI tool，目前docker引擎内部也是基于runc构建的。
2019年2月11日，研究人员通过oss-security邮件列表披露了runc容器逃逸漏洞的详情，根据OpenWall的规定EXP会在7天后也就是2019年2月18日公开。

此漏洞允许以root身份运行的容器以特权用户身份在主机上执行任意代码。实际上，这意味着容器可能会破坏Docker主机(覆盖runc CLI)。
所需要的只是容器中允许使用root。攻击者可以使用受感染的Docker镜像或对未受感染的正在运行的容器运行exec命令。针对此问题的已知缓解措施包括：

- 使用只读主机文件系统运行
- 运行用户命名空间
- 不在容器中运行root
- 正确配置的AppArmor / SELinux策略（当前的默认策略不够）

收到披露邮件后，RancherOS团队尝试编写了攻击脚本，在一个普通容器中运行一个非常简单的脚本就完成了对主机的攻击，将主机上的runc替换成了其他程序。

Docker在第一时间发布了[18.09.2](https://github.com/docker/docker-ce/releases/tag/v18.09.2)，用户可升级到此版本以修复该漏洞。
但是通常由于各种因素，用户的生产环境并不容易升级太新的Docker版本，Rancher Labs已经将修复程序反向移植到所有版本的Docker。
相关修补程序以及安装说明，请参考[rancher/run-cve](https://github.com/rancher/runc-cve)。

RancherOS作为一款容器化操作系统，其中很多组件依赖runc，我们也在第一时间更新了补丁并发布了v1.5.1和v1.4.3两个版本。

### RancherOS的更新  

RancherOS的核心部件system-docker和user-docker都依赖runc，所以v1.5.1和v1.4.3都对他们进行了更新。而针对user-docker，RancherOS可以切换各种版本的docker engine，
所以我们对一下docker engine都进行了反向移植：**v1.12.6/v1.13.1/v17.03.2/v17.06.2/v17.09.1/v17.12.1/v18.03.1/v18.06.1**。

默认安装v1.5.1和v1.4.3，补丁程序已经是内置的，你无需任何操作就可以避免该漏洞。如果你希望使用早起的docker版本，那么切换user-docker时，请使用上面提到的补丁修复版本：

```
root@ip-172-31-2-241:~# ros engine list
disabled docker-1.12.6
disabled docker-1.13.1
disabled docker-17.03.1-ce
disabled docker-17.03.2-ce
disabled docker-17.06.1-ce
disabled docker-17.06.2-ce
disabled docker-17.09.0-ce
disabled docker-17.09.1-ce
disabled docker-17.12.0-ce
disabled docker-17.12.1-ce
disabled docker-18.03.0-ce
disabled docker-18.03.1-ce
disabled docker-18.06.0-ce
enabled  docker-18.06.1-ce
disabled docker-18.09.0
disabled docker-18.09.1
disabled docker-18.09.2

root@ip-172-31-2-241:~# ros engine switch docker-17.03.2-ce
...

root@ip-172-31-2-241:~# docker info | grep Server
Server Version: 17.03.2-ce
```

同时v1.5.1版本也是支持docker 18.09.2，你可以切换到该版本，如果你考虑使用Docker官方的修复版本。只需简单运行: `ros engine switch docker-18.09.2`。

### v1.5.1 和 v1.4.3  

推荐您使用最新的v1.5.1版本，除了修复CVE-2019-5736漏洞外还支持其他新特性以及一些Bug Fix。当然，仍然有很多用户在使用1.4.x版本，所以我们也发布了v1.4.3，
它只修复了runc漏洞，没有其他额外的更新。

AWS相关镜像已经上传到各个region中，可以直接搜索查找并使用，包括AWS中国区。其他主要镜像列表参考：https://github.com/rancher/os/blob/v1.5.x/README.md#release

更多新特性和Bug Fix请参考v1.5.1的[Release Notes](https://github.com/rancher/os/releases/tag/v1.5.1)

文档说明：https://rancher.com/docs/os/v1.x/en/

最后，RancherOS还是一个小众的开源项目，我们专注Docker在Linux上的精简体验，如果喜欢RancherOS，请下载使用，非常期待收到您的反馈。
同时Github上的star，也是鼓励我们继续前行的精神动力。
