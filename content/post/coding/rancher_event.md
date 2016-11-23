+++
menu = ""
banner = ""
images = [
]
date = "2016-08-25T13:44:04+08:00"
title = "Rancher event机制及其实践指南"
slug = "rancher-envent"
tags = [
    "Rancher",
]
categories = [
    "coding",
]
description = ""

+++
Rancher在v1.0版本为了Kubernetes的集成，开始扩展并逐步完善了Rancher event机制，
event机制可以完成技术架构上的解耦。那么Rancher都提供了哪些event？
Rancher内部是如何使用的？以及我们如何用event来增强一些定制服务？
<!--more-->

### 引言
我们的Rancher官方技术社区已经创立些许时日了，相信通过我们的线下meetup和线上布道工作，
很多朋友对Rancher的使用已经掌握得很纯熟了。一些高级用户开始真正把自己的业务进行微服务化并向Rancher迁移，
在迁移的过程中，由于业务本身的复杂性特殊性，
可能需要利用Rancher的一些高级特性甚至要对Rancher进行一定的扩展，
这就需要对Rancher的一些组件的实现机制有些许了解。

本次分享就介绍一下Rancher的event机制，由于相关内容文档极其欠缺，本人也只是通过实践和代码阅读分析其原理，
如有谬误欢迎交流指正，小腩双手红包奉上！同时为保证收视率，本次分享会以原理+实践的方式进行。

### 原理分析
在大规模系统架构中，event机制通常采用消息驱动 ，它对提升分布式架构的容错性灵活性有很大帮助，
同时也是各个组件之间解耦的利器。Rancher能够管理N多的agent同时又拆分出各种服务组件，
event机制是必不可少的。为实现event机制，通常我们会采用RabbitMQ、ActiveMQ、ZeroMQ等中间件来实现。
而Rancher则采用了基于websocket协议的一种非常轻量级的实现方式，
它的好处就是极大程度的精简了Rancher的部署，Rancher无需额外维护一个MQ集群，
毕竟websocket的消息收发实现是非常简单的，各种语言库均可以支持。

这里我们会考虑一个问题，websocket毕竟不是真正工业级MQ的实现，消息不能持久化，
一旦某个event的处理出现问题，或者发生消息丢失，Rancher如何保证各个资源的原子性一致性？
Rancher中有一个processpool的概念，它可以看做一个所有event的执行池，
当API/UI/CLI有操作时，Rancher会把操作分解成多个event并放入processpool中。
比如删除一个容器时会把 compute.instance.remove 放入processpool中，
这个event会发送到对应的host agent上，agent处理完成后会发送reply给rancher-server。
如果在这个过程中，由于网络问题消息丢失，或者agent上执行出现问题，
rancher-server没有收到reply信息，cattle会把这个event重新放到processpool中再次重复上面的过程，
直到 compute.instance.remove 完成操作，这个容器的状态才会在DB中更新，
否则该容器状态会一直处于lock不能被其他服务更新。当然cattle不会把这些event不停的重复执行下去，
通常会设置一下TIMEOUT超出后便不再执行（有些资源没有TIMEOUT机制）。

上面的表述，我们其实可以在UI上看到这个过程，RancherUI上的Processes的Running Tab页上就能实时得看到这些信息，
Processes 在排查一些Rancher的相关问题是非常有用的，大家可以养成 ”查问题先查Processes“的好习惯：  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa1zux9y7bj20ly091dh6.jpg)  
那么监听event的URL怎么设定呢？非常简单：  
`ws://<rancher-server-ip>:8080/v1/projects/<projectId>/subscribe?eventNames=xxxx`  
除此之外还需要加上basic-auth的header信息：  
`Authorization: Basic +base64encode(<cattle-access-key>:<cattle-secret-key>)`  
如果是Host上的agent组件，除此之外还需要添加agentId参数：  
`ws://<rancher-server-ip>:8080/v1/projects/<projectId>/subscribe?eventNames=xxxx&agentId=xxxx`  
agentId 是注册Host时生成的，如果没有agentId参数，
任何有关无关的event都会发送到所有的Host agent上，这样就会发生类似“广播风暴”的效果。

Host agent上运行很多组件，其中python-agent是负责接收和回执event信息的，
其运行日志可以在Host上的/var/log/rancher/agent.log文件中查看。
细心的朋友可能会有疑问，我们在添加Host时执行agent容器时并没有指定cattle-access-key和cattle-secret-key，
也就是说python-agent运行时如何获取这两个秘钥信息呢？

其实Rancher有两种apikey：一种是我们熟知的在UI上手动创建的apikey；
另外一种就是agentApikey，它是系统级的，专门为agent设定，
添加Host时会先把agentApikey发送到Host上。在cattle的credential表中可以查询到相关信息：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa1zxokn7dj20ea05774u.jpg)

eventNames都定义了哪些呢？下面两个文件可以参考：  
* [IaasEvents.java](https://github.com/rancher/cattle/blob/master/code/iaas/events/src/main/java/io/cattle/platform/iaas/event/IaasEvents.java
)
* [系统级的event定义](https://github.com/rancher/cattle/blob/master/code/packaging/app-config/src/main/resources/META-INF/cattle/process/spring-process-context.xml
)，详细到每种资源(host、volume、instance、stack、service等)的event定义。

此外，我们在UI/CLI/API上的某个操作都会被分解成多个event来执行，
每个event信息都会被保存在mysql中，每个event执行成功后会设置成purged状态，
所以记录并不会真正删除，这就会导致相应的表（container_event表、service_event表、process_instance表）会无限膨胀下去。
Rancher为解决此问题提供了周期性清理机制：  

1. events.purge.after.seconds 可以清理container_event和service_event，每两周清理一次
2. process_instance.purge.after.seconds可以清理process_instance，每天清理一次。  
这两个配置我们都可以在<http://rancher-server-ip:8080/v1/settings>动态修改。

### 实践操作
下面我们来实践一下，看看如何在程序中实现对Rancher event的监听。
Rancher提供了resource.change事件，这个事件是不用reply的，就是说不会影响Rancher系统的运行，
它是专门开放给开发者用来实现一些自己的定制功能，所以我们就以resource.change作为例子实践一下。

Rancher的组件大部分都是基于Golang编写，所以我们也采用相同的语言。
为了能够快速实现这个程序，我们需要了解一些辅助快速开发的小工具。  

* Trash，Golang package管理的小工具，可以帮助我们定义依赖包的路径和版本，非常轻量且方便；
* Dapper，这个是基于容器实现Golang编译的工具，主要是可以帮助我们统一编译环境；
* Go-skel，它可以帮我们快速创建一个Rancher式的微服务程序，可以为我们省去很多的基本代码，
  同时还集成了Trash和Dapper这两个实用的小工具。

首先我们基于go-skel创建一个工程名为 scale-subscriber （名字很随意），执行过程需要耐心的等待：  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa2044jpu9j20im0ce78n.jpg)

执行完毕后，我们把工程放到GOPATH中，开始添加相关的逻辑代码。
在这之前我们可以考虑添加一个healthcheck的服务端口，纵观Rancher所有的微服务组件，
基本上除了主程序之外都会添加 healthcheck port，这个主要是为了与Rancher中的healthcheck功能配合，
通过设定对这个端口的检查机制来保证微服务的可靠性。我们利用Golang的goroutine机制，
分别添加主服务和healthcheck服务：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa2053h2k7j20i908u0uu.jpg)

主服务的核心就是监听resource.change的信息，同时注册handler来获取event的payload信息，
从而可以定制扩展自己的逻辑，这里需要利用Rancher提供的
[event-subsriber](https://github.com/rancher/event-subscriber)库来快速实现。
如下图，在**reventhandlers.NewResourceChangeHandler().Handler**中就可以实现自己的逻辑：  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa206sahqtj20ka07habz.jpg)

这里我们只是演示监听event的机制，所以我们就不做过多的业务逻辑处理，打印输出event信息即可。
然后在项目根目录下执行make，make会自动调用dapper，在bin目录下生成scale-subscriber，
我们执行scale-subscriber就可以监听resource.change的信息：  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa207fymwlj20u904gdhf.jpg)

这里我们可以看到分别启动了healthcheck功能和event listener。然后可以在UI上随意删除一个stack，
scale-subsciber就可以获取event信息：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa207xudcej20ni07mgpy.jpg)

### 附加小技巧
如要将其部署在Rancher中，那就可以将scale-subsciber执行程序打包放到镜像中，通过compose来启动。
但是我们知道scale-subsciber启动需要指定CATTLE_URL、CATTLE_ACCESS_KEY、CATTLE_SECRET_KEY，
正常的做法就是我们需要先生成好apikey，然后启动service的时候设置对应的环境变量。
这样做就会有弊端，就是apikey这种私密的信息不得不对外暴露，而且还要手动维护这些apikey，非常不方便。

Rancher提供了一个非常方便的做法，就是在service中添加两个label，
**io.rancher.container.create_agent: true**和**io.rancher.container.agent.role: environment**，
设定这两个label后，Rancher引擎会自动创建apikey，并把相应的值设置到容器的ENV中，
只要你的程序通过系统环境变量来读取这些值，就会非常顺利的运行起来。

