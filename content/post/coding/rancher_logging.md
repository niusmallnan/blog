+++
date = "2017-04-11T11:56:14+08:00"
slug = "rancher-logging"
title = "Rancher体系下容器日志采集"
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

+++
在Rancher体系内，实现容器日志采集。
<!--more-->
### 引言
一个完整的容器平台，容器日志都是很重要的一环。尤其在微服务架构大行其道状况下，程序的访问监控健康状态很多都依赖日志信息的收集，
由于Docker的存在，让容器平台中的日志收集和传统方式很多不一样，日志的输出和采集点收集和以前大有不同。
本文就探讨一下，Rancher平台内如何做容器日志收集。

### 当前现状
纵览当前容器日志收集的场景，无非就是两种方式：一是直接采集Docker标准输出，容器内的服务将日志信息写到标准输出，
这样通过Docker的log driver可以发送到相应的收集程序中；二是延续传统的日志写入方式，容器内的服务将日志直接写到普通文件中，
通过Docker volume将日志文件映射到Host上，日志采集程序就可以收集它。

第一种方式足够简单，直接配置相关的Log Driver就可以，但是这种方式也有些劣势：

1. 当主机的容器密度比较高的时候，对Docker Engine的压力比较大，毕竟容器标准输出都要通过Docker Engine来处理。
2. 尽管原则上，我们希望遵循一容器部署一个服务的原则，但是有时候特殊情况不可避免容器内有多个业务服务，
这时候很难做到所有服务都向标准输出写日志，这就需要用到前面所说的第二种场景模式。
3. 虽然我们可以先选择很多种Log Driver，但是有些Log Driver会破坏Docker原生的体验，比如docker logs无法直接看到容器日志。

基于以上考虑，一个完整的日志方案必须要同时满足标准输出和日志卷两种模式才可以。当然完整的日志体系中，并不仅仅是采集，
还需要有日志存储和UI展现。日志存储有很多种开源的实现，这个一般用户都会有自己钟情的选择，而UI展现更是各家有各家的需求，
很难形成比较好的标准，一般都是通过定制化方式解决。所以此文主要展现的方案是日志采集方案，当然在存储和UI展现上会对接开源实现，
没有特殊需求的情况下，也可以拥有一个完整的体验。  
![](http://ww1.sinaimg.cn/large/006tNc79ly1feiptl1cx3j31js0v6401.jpg)

### Rancher下的解决方案
如上面图中所示，日志存储和UI展现可以直接使用ElasticSearch & Kibana，日志采集方面如之前所分析，需要对接两种采集模式，
所以这部分采用Fluentd & Logging Helper的组合，Fluentd是很通用的日志采集程序，拥有优异的性能，相对Logstash来说同等压力下，
其内存消耗要少很多。Logging Helper是可以理解微Fluentd的助手，它可以识别容器内的日志卷文件，通知Fluentd进行采集。
此外，由于要保证Dokcer和Rancher体验的完整性，在Docker Log Driver的选型上支持json-file和journald，其原因：
一是json-file和journald相对来说比较常用；二是这两种驱动下，docker logs依然可以有内容输出，保证了体验的完整性。

下面开始说明，整个方案的部署过程。先用一张图来描述整体的部署结构，如下：  
![](http://ww3.sinaimg.cn/large/006tNc79ly1feirbhlr0ij31700j4aaj.jpg)  
总共划分三个ENV，其中一个ENV部署ES & Kibana，另外两个ENV分别添加json-file驱动的主机和journald驱动的主机。
详细部署过程如下：

创建三个ENV，使用Cattle引擎。设置Logging Catalog方便部署，
在Admin--Settings页面中添加Catalog，地址为：<https://github.com/niusmallnan/rancher-logging-catalog.git>  
![](http://ww1.sinaimg.cn/large/006tNc79ly1feirqbal91j31du0b2q3d.jpg)

进入ES-Kibana ENV中，部署ElasticSearch & Kibana，这两个应用在Community Catalog中均可以找到，部署非常简单，
需要注意的是，建议选择Elasticsearch 2.x，Kibana中的Elasicsearch Source选择elasticseach-clients：  
![](http://ww2.sinaimg.cn/large/006tNc79ly1feirvmhq6qj30sc0mimxt.jpg)

分配两台主机并安装Docker，其中Log Driver分别选择json-file和journald，并将主机添加到各自的ENV中。
在这两个ENV中添加External Link指向之前部署的Elasticsearch地址：  
![](http://ww1.sinaimg.cn/large/006tNc79ly1feis5geevwj319e0gmaas.jpg)  
然后在Jsonfile & Journald ENV中添加Rancher Logging应用，打开对应的catalog，ES link指向刚才设定的External link，
其中Volume Pattern是日志卷的匹配模式，File Pattern是日志卷内的日志文件匹配模式：  
![](http://ww2.sinaimg.cn/large/006tNc79ly1feis130yinj31ke0scwfk.jpg)

以上部署完成之后，部署一些应用并产生一些访问日志，就可以在Kibana的界面中看到：  
![](http://ww4.sinaimg.cn/large/006tNc79ly1feistxqn3ej31a60x0jto.jpg)  

若要使用日志卷方式，则需要在service启动的时候配置volume，volume name需要匹配之前设定的Volume Pattern：  
![](http://ww3.sinaimg.cn/large/006tNc79ly1feisvr5tlkj317m0ju3zd.jpg)

秉承一切开源的原则，相关实现可以查看一下链接：

1. <https://github.com/niusmallnan/rancher-fluentd-package>
2. <https://github.com/niusmallnan/logging-helper>
3. <https://github.com/niusmallnan/rancher-logging-catalog>

### 总结
通过Fluentd我们可以对接很多第三方日志存储体系，但是Fluentd自身并不能完成日志采集的所有场景，所以非常需要Logging Helper的帮助。
通过Logging Helper可以定制出一些额外采集规则，比如可以过滤某些容器日志等等。当然真正大规模生产环境日志平台，其实是对整个运维体系的考验，
单纯靠开源程序的组合并不能真正解决问题。



