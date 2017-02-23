+++
date = "2017-02-23T17:33:56+08:00"
title = "关于Subcribe Rancher Events的思考"
slug = "thinking-about-subcribe-rancher-events"
description = ""
menu = ""
banner = ""
images = [
]
tags = [
    "Rancher",
]
categories = [
    "coding",
]

+++
路漫漫其修远兮，吾将上下而求索
<!--more-->
### 引言
几乎每个大型的分布式的集群软件，都离不开一样东西，就是所谓的message bus（消息总线），
它就如同人体的血管一样，连通着各个组件，相互协调，一起工作。
与很多同类软件不同的是，Rancher使用的基于websocket协议实现的消息总线，
Rancher不会依赖任何MQ，基于websocket的实现十分轻量级，
同时在各种语言库的支持上，也毫无压力，毕竟websocket是HTTP的标准规范之一。

### 初识Rancher Events
基于websocket的消息总线可以很好的与前端兼容，让消息的传递不再是后端的专利。
在Rancher UI上，很容易就能捕获到rancher events，比如：  
![](https://ww4.sinaimg.cn/large/006tNc79ly1fd0k5tuin9j30it07at9x.jpg)  
这里面监听的事件名称是resource.change，这个resource.change在前端UI上有很大的用处，
其实我们都知道，很多POST形式的create请求并不是同步返回结果的，因为调度引擎需要处理，
这个等待的过程中，当然不能前端一直wait，所以做法都是发起create后直接返回HTTP 202，
转入后台执行后，Rancher的后端会把创建的执行过程中间状态不断发送给消息总线，
那么前端通过监听resource.change就会获得这些中间状态，这样在UI上就可以给用户一个很好的反馈体验。

当然Rancher Events并不是只有resource.change，比如在Iaas Events集合中就有如下这些：  
![](https://ww1.sinaimg.cn/large/006tNc79ly1fd0kccvpwxj30gk0ac40k.jpg)  

除了Events的事件定义，当然还有如何去subscribe 这些events，这部分内容我之前的文章中有所涉猎，
便不赘言。<http://niusmallnan.com/2016/08/25/rancher-envent/>

### Subscribe Rancher Events的架构模式
Rancher的体系内，很多微服务的组件都是基于Subcribe Rancher Events这种架构，举个例子来看，
以rancher-metadata组件为例：  
![](https://ww4.sinaimg.cn/large/006tNc79ly1fd0kmwbmu4j30iy07wq3n.jpg)  
metadata服务可以提供当前host的元数据查询，我们可以很容器的知道env内的stack/service/container的情况，
这些数据其实由rancher-server也就是cattle引擎生成的，那么生成之后怎么发送给各个agent呢？
其实就是metadata进行了subcribe rancher events，当然它只监听了config.update事件，
只要这个事件有通知，metadata服务便会下载新的元数据，这样就达到了不断更新元数据的目的。

随着深入的使用Rancher，肯定会有一些伙伴需要对Rancher进行扩展，那就需要自行研发了，
毕竟常见的方式就是监听一些事件做一些内部处理逻辑，并在DB中存入一些数据，
同时暴露API服务，架构如下：  
![](https://ww3.sinaimg.cn/large/006tNc79ly1fd0kt6rttwj309o072jrn.jpg)

如果需要做HA，可能需要scale多个这样的服务，那么架构就变成这样：  
![](https://ww2.sinaimg.cn/large/006tNc79ly1fd0ku7zaafj30g30avdgq.jpg)  
这里其实会有一个问题，如果你监听了一些广播事件，那么实际上每个实例都会收到同样的事件，
那么你的处理逻辑就要注意了，尤其是在处理向DB中写数据时，一定要考虑到这样的情况。

比如，可以只有其中一个实例来监听广播事件，这样不会导致事件重复收取：  
![](https://ww4.sinaimg.cn/large/006tNc79ly1fd0kxdd6vmj30g30b1dgm.jpg)  
Event Handler要考虑一定failover机制，这样事件收取不会长时间中断。

Rancher Events有一些非广播事件，那么就需要在subscribe的时候指定一些特殊参数，
这样事件就会只发送给注册方，不会发送给每个节点，比如：  
![](https://ww3.sinaimg.cn/large/006tNc79ly1fd0kztt3k6j30fx0audgr.jpg)

### 总结
此文算是这段时间做Rancher服务扩展的心得，深度参与一个开源软件最终肯定会希望去改动它扩展它。
这也是客观需求所致，开源软件可以拿来即用，但是真正可用实用，必须加以改造，适应自身需求。

