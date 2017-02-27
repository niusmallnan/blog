+++
images = [
]
tags = [
    "Docker",
]
categories = [
    "coding",
]
description = ""
date = "2016-12-27T07:15:44+08:00"
title = "Docker中的device or resource busy问题分析"
slug='docker-device-resource-busy'
menu = ""
banner = ""

+++
拯救一脸懵逼
<!--more-->
### 现象描述
使用docker的时候，当我们删除某些容器，有时候会报出device or resource busy错误：
```
Error response from daemon:
Unable to remove filesystem for
a9a1c11e8210d60ddba09f95ea93ae21f32327c4e5877c218862c752d1088533: 
remove /var/lib/docker/containers/a9a1c11e8210d60ddba09f95ea93ae21f32327c4e5877c218862c752d1088533/shm:
device or resource busy
```
然后可以查看相关容器的状态变成了很少见的Dead状态。

### 原因分析
通常我们看到device or resource busy首先想到的是，这个设备被其他程序占用导致。
在容器中，其实理论是一样的，每个容器生成都会有一个containers/<uuid>/shm设备产生，
恰巧这个设备被其他程序mount了，就会被占用无法卸载，也就是device or resource busy。

那么什么情况下会占用containers/<uuid>/shm设备呢？其最大的可能原因就是容器启动时挂载了/var/lib/docker目录，
此时恰巧docker出现了bug，顺带mount了containers/<uuid>/shm设备。
为证实猜想，我们启动一个挂载/var/lib/docker目录的容器，然后到容器中查看/proc/mounts，如下：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fb4znsftd3j30kh08ntfm.jpg)  
其实除了/var/lib/docker做了多余挂载之外，映射/var/run和/run/目录都会有同样的问题：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fb4zo5bmavj30ex04n0u4.jpg)

### 修复方式
首先这肯定是docker的一个bug，很明显它不应该做多余的mount。这个bug在docker v1.12.4版本做了一些修复，
相关PR<https://github.com/docker/docker/pull/29083>，类似的issue<https://github.com/docker/docker/issues/20560>，
都可以在github上找到。所以docker v1.12.4之前也都会比较容易碰到这个问题，
虽然有一些FIX已经merge了，但是其实并没有完全解决，起码我在docker v1.12.6上使用AUFS驱动，
还是会发现这个问题。

那么如果因为各种原因无法变更docker，但是还想避免这个问题怎么办？
比如可以在mount /var/lib/docker（或是你迁移之后的DockerRootDir）的容器中执行以下脚本：
```
#!/bin/bash

for i in $(curl -s --unix /var/run/docker.sock http://localhost/info | jq -r .DockerRootDir) /var/lib/docker /run /var/run; do
    for m in $(tac /proc/mounts | awk '{print $2}' | grep ^${i}/); do
        umount $m || true
    done
done
```
主要目的是把多余mount的path卸载掉，这样就不会因为同时挂载导致device or resource busy的问题。
需要注意的是，是否也要umount /run和/var/run要结合自身实际应用去考虑。

