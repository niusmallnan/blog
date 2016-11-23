+++
tags = [
    "Rancher",
    "Kubernetes",
]
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""
images = [
]
date = "2016-07-28T14:49:47+08:00"
title = "Kubernetes in Rancher1.0 架构分析"
slug = "k8s-in-rancher-1-0"

+++
该主题是本人于Rancher k8s 北京meetup进行的一次线下分享，应Rancher China官方之邀，
重新梳理成文字版本，便于大家阅读传播，如有问题纰漏或任何不妥之处随时联系牛小腩（niusmallnan），
我会以最快速度更正。
<!--more-->
### 引言
在Rancher 1.0版本开始，Rancher逐步增加了Kubernetes、Swarm、Mesos等多编排引擎的支持，
很多朋友就此产生了疑惑，诸如Cattle引擎和这几个之间到底什么关系？
每种引擎是如何支持的？自家的业务环境如何选型？我们将逐步揭开这些神秘面纱，
了解基础架构才能在遇到问题时进行有效的分析，进而准确的定位问题并解决问题，
因为没有一种生产环境是完全可靠的。基于这个背景下，这次我们首先向大家介绍kubernetes in Rancher的架构。

### 分析
从现在Rancher的发展节奏来看，Cattle引擎已经被定义成Rancher的基础设施引擎，
而Rancher的基础设施服务都包括哪些呢？如下：

* Networking，rancher的统一网络服务，由rancher-net组件提供
* Load Balancer，rancher的负载均衡服务，目前来看套路基本上是基于Haproxy来构建
* DNS Service，rancher的dns服务，主要是为了提供服务发现能力，由rancher-dns组件来提供
* Metadata Service，rancher的元数据服务，metadata是我们通过compose编排应用时的利器，
  可以很灵活的像service中注入特定信息
* Persistent Storage Service，持久化存储服务目前是由convoy来提供，
  而对于真正的后端存储的实现rancher还有longhorn没有完全放出
* Audit Logging，审计日志服务是企业场景中比较重要的一个属性，
  目前是集成在cattle内部没有被完全分离出来

所以Rancher在接入任何一种编排引擎，最终都会把基础设施服务整合到该引擎中，
Kubernetes in Rancher的做法正是如此。

Kubernetes各个组件的角色可以归为三类即Master、Minion、Etcd，
Master主要是kube-apiserver、kube-scheduler、kube-controller-manager，
Minion主要是kubelet和kube-proxy。Rancher为了融合k8s的管控功能，
又在Master中添加了kuberctrld、ingress-controller、kubernetes-agent三个服务来打通Rancher和K8s，
同时每个node上都会依赖Rancher提供的rancher-dns、rancher-metadata、rancher-net这些基础设施服务。  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa21xnsfbrj20p40c0wgm.jpg)

由于K8s是基于Cattle引擎来部署，所以在K8s在部署完成之后，
我们可以通过Link Graph来很清晰的看到整体的部署情况。  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa21y87258j20mi07tdh0.jpg)  
整个服务基于Cattle引擎的rancher-compose构建，新增节点后自动添加kubelet和kube-proxy服务
（此处利用了Global Label的特性），各个组件都添加了health-check机制，
保证一定程度的高可用。考虑到Etcd最低1个最多3个节点，
单台agent host就可以部署K8s，三节点agent host则更合理些。

K8s集群完成部署后，我们就可以在其中添加各种应用服务，
目前Rancher支持管理K8s的service、pod、replication-controller等，
我们可以用一张图来形象得描述一下应用视图结构。  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa21z5ellcj20qj0cuq5u.jpg)  
rancher-net组件会给每个pod分配一个ip，rancher-dns则替代了K8s的Skydns来实现服务发现，
在pod的容器内部依然可以访问rancher-metadata服务来获取元数据信息。除了这三个基础服务外，
我们之前提到的kuberctrld、ingress-controller、kubernetes-agent也在其中扮演者重要角色。

无论是K8s还是Rancher，其中一些抽象对象（如rancher的stack/service，或者K8s的serivice/pod）
在属性更新时都会有events产生，在任何服务入口来更改这些抽象对象都会有events产生，
所以要保证Rancher和k8s能够互相感知各自对象的更新，那么kubernetes-agent就应运而生了。  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa22166xnyj20nt0d60ut.jpg)  
诸如K8s的namespaces、services、replicationcontrollers、pods等对象的信息变更会及时通知给Rancher，
而Cattle管理的Host资源出现信息变更（诸如host label的变动）也会通知给K8s。

简单得说kubernetes-agent是为了维护Rancher和K8s之间的对象一致性，
而真正要通过Rancher来创建K8s中的service或者pod之类的对象，
还需要另外一个服务来实现，它就是kubectrld，直观的讲它就是包装了kubectrl，
实现了其中kubectl create/apply/get 等功能。  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa221z612jj20o80bhq4q.jpg)  
所有的K8s对象创建请求都会走cattle引擎，cattle会把请求代理到kubectrld启动的一个api服务。
除此之外，还会监听rancher events来辅助实现相关对象的CRUD。

K8s上创建的service如需对外暴露访问，那么必然会用到LoadBalancer Type和Ingress kind，
注意K8s概念下的LoadBalancer和Ingress略有不同，LoadBalancer的功能主要关注在L4支持http/tcp，
而Ingrees则是要实现L7的负载均衡且只能支持http。K8s 的LoadBalancer需要在K8s中实现一个Cloud Provider，
目前只有GCE，而Rancher则维护了自己的K8s版本在其中提供了Rancher Cloud Provider。
对于Ingress则是提供了Ingress-controller组件，它实现了K8s的ingress框架，
可以获取ingress的创建信息并执行相应的接口。当然最终这两者都会调用cattle api来创建Rancher的负载均衡，
且都是通过Haproxy完成负责均衡功能。  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa22370rzfj20hv0b6gn5.jpg)

### 后语
以目前K8s社区的火热势头，Rancher应该会持续跟进并不断更新功能优化架构，
待到Rancher1.2发布之后，CNI的支持会是一个里程碑，
到那时Kubernetes in Rancher也会更加成熟，一起向着最好用Kubernetes发行版大踏步的前进。


