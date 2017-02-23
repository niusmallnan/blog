+++
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""
images = [
]
tags = [
    "Rancher",
]
date = "2017-02-23T18:58:19+08:00"
title = "Rancher如何按计算资源调度"
slug = "rancher-scheduler-based-on-resource"

+++
非常简单的说明一下，Rancher如何做按计算资源调度。
<!--more-->
### 引言
按计算资源调度基本上是各大编排引擎的标配，Rancher在v1.2版本后也推出了这个功能，
很多朋友并不知道，主要是因为当前的实现还并不是那么智能，故不才欲写下此文以助视听。

### 实现机制
Rancher的实现比较简单，其主要是通过Infra services中的scheduler服务来实现，整体的逻辑架构如下：  
![](https://ww3.sinaimg.cn/large/006tNc79ly1fd0lxplni2j30m90a7q49.jpg)  
scheduler会订阅Rancher Events，主要是scheduler相关事件，当有调度需求时候，scheduler就会收到消息，
通过计算将合适的调度目标返回给cattle。比如说现在支持memory和cpu为基准，
那么scheduler会不断根据metadata的数据变化来计算资源的使用量，最后可根据资源剩余量为调度目标排序，
这样就可以完成按计算资源调度的目标。

之前有说，Rancher的实现并不智能，这在于在计算资源使用量的时候，Rancher并不是通过一套复杂数据采集机制来计算，
而是通过用户在创建service的时候标注reservation的方式，这个地方很多朋友并没有注意到：  
![](https://ww4.sinaimg.cn/large/006tNc79ly1fd0m3wzn0dj30lf08f3ze.jpg)

除此之外，在每个节点的资源总量上也是可配置的，我们完全可以进行一个整体预留的设置，比如：  
![](https://ww4.sinaimg.cn/large/006tNc79ly1fd0m5p3spuj30f90amaao.jpg)

### 总结
这个实现看似简单，其实这是提供了一个很好的扩展能力。如果我们有自己的监控采集体系，
完全可以在scheduler的时候调用我们自身监控接口来计算资源，这样就能达到我们所认可的“智能”了。
