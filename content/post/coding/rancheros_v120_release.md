+++
title = "RancherOS v1.2.0发布"
slug = "rancheros-v120-release"
banner = ""
images = [
]
tags = [
    "RancherOS",
]
categories = [
    "coding",
]
description = ""
menu = ""
date = "2018-02-07T16:52:31+08:00"

+++
一言不合就发新版本
<!--more-->

RancherOS v1.2.0版本于北京时间2月7日正式发布，从v1.1到v1.2开发周期中，我们收集到了社区用户和商业用户的Bug report和Feature request，
感谢大家为此作出的贡献。这个周期内，Meltdown和Spectre漏洞曝出给OS界造成了沉重的打击，我
们也时刻紧跟业界动向，第一时间把漏洞补丁更新到RancherOS中。

### Spectre Var.2 漏洞修复  
对于Spectre变种2，我们采用了新的GCC编译器开启Retpoline指令重新编译了内核。而Intel的微码补丁，我们并没有采用，因为业界对这个补丁诟病很深，
已经造成了很多云厂商的Crash问题。

基于最新的RancherOS v1.2.0可以使用这种方式检测Meltdown和Spectre的修复状态：

```
sudo system-docker run --rm -it -v /:/host niusmallnan/spectre-meltdown-checker
```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fo80fauf1fj319s0v8gqy.jpg)

### 不再支持ARM32  
考虑到32位操作系统发展情况，以及其他Linux发行版的支持情况，不再对ARM 32位进行硬件支持。
ARM 64位依然会继续支持，你依然可以继续在合适的树莓派型号上使用它。

树莓派镜像：https://github.com/rancher/os/releases/download/v1.2.0/rancheros-raspberry-pi64.zip
安装方式参考：https://www.raspberrypi.org/documentation/installation/installing-images/

### 各个Console更新  
RancherOS支持多种console，比如Ubuntu、Debian、Alpine等等，默认使用busybox console。
Busybox console基于[Buildroot](https://github.com/buildroot/buildroot)，本次发布更新了较新的稳定版本。

其他console均更新成LTS版本，版本如下：

- Alpine: 3.7
- CentOS: 7.4.1708
- Debian: stretch
- Fedora: 27
- Ubuntu: xenial

### 其他  
AWS相关镜像已经上传到各个region中，可以直接搜索查找并使用，AWS中国区目前还不支持。其他主要镜像列表：

- 阿里云镜像：https://github.com/rancher/os/releases/download/v1.2.0/rancheros-aliyun.vhd
- GCE镜像：https://github.com/rancher/os/releases/download/v1.2.0/rancheros-gce.tar.gz
- OpenStack镜像：https://github.com/rancher/os/releases/download/v1.2.0/rancheros-openstack.img

更多新特性和Bug Fix请参考v1.2.0的[Release Notes](https://github.com/rancher/os/releases/tag/v1.2.0)

文档说明：http://rancher.com/docs/os/v1.2/en/

