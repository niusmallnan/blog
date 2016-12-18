+++
menu = ""
banner = ""
images = [
]
tags = [
    "Rancher",
    "Rancher v1.2"
]
categories = [
    "coding",
]
description = ""
date = "2016-12-14T21:53:27+08:00"
title = "Rancher v1.2 Swarmkit的实现"
slug = "rancher12-swarmkit-architect"

+++
Rancher v1.2系列文章，已独家授权Rancher Labs，转载请联系Rancher China相关人员。
<!--more-->
### 引言
Rancher v1.2更新了之前对Swarm的支持，与Docker一样抛弃了就有的Swarm，
选择支持Swarmkit。Swarmkit引擎非常轻量级，由于其内置早Docker Engine中，
所以部署起来会非常方便。虽然目前Swarmkit引擎还在不断发展，而且bug也很多，
但是它也有其擅长的使用场景，比如简单的CI/CD场景，它会非常灵活简洁。
本文将带大家体验一下，Rancher v1.2对Swarmkit的支持。

### 部署与使用
部署方面秉承Rancher一贯的原则，非常简单，只需要在创建Env时选择Sawrm即可。  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1favae6x30jj30nn09odgr.jpg)  
Env创建完毕后，会看到多个Infra Service需要创建，这时候和其他引擎一样，
我们需要向Env中添加Host。我们知道Swarm的node有两种：Manager和Worker。
Rancher创建的Swarm集群默认是3个Manager，多个Manager内有一个是Leader，
另外两个备用。这样单个Host出问题，新的Leader会很快选举出来，保证集群的稳定性。
比如我添加了两个Host，默认是先添加Manager角色，
所以2个Host都会以Manager方式添加，如下图所示：  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1favaeq4hrhj30hc074aav.jpg)  
进入其中一台Host内，查看swarm集群状态，可以看到一个是Leader，另外一个Reachable做备用。  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1favaey8g8qj30fc01wq3g.jpg)  
尝试创建一个简单的程序，查看与UI上的联动效果，如图所示：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1favaf8yytqj30eb05vdgv.jpg)  
如果使用Swarmkit的自定义网络方式，情况如何？虽然在UI上显示无IP，
但是进入容器内部可以看到overlay对应的网卡，如图所示：
![](http://ww3.sinaimg.cn/large/006tKfTcjw1favafscwv8j30ii07habk.jpg)

### 实现原理
那么Rancher是如何来完成Swarmkit的部署和联动呢？Rancher中Swarmkit也是基于Cattle来部署的，
根据之前的文章分析，我们可以知道Rancher的基础设施编排的定义都是通过catalog中的infra-templates实现的，
Swarmkit比较特殊它是在community-catalog中定义的，如果一直在rancher-catalog中寻找肯定找不到。
compose文件中定义了一个service swarmkit-mon，如图所示：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1favaggy2snj30iz0at40g.jpg)  
如果探究原理，我们就需要知道swarmkit-mon对应的镜像是如何定义的。
rancher/swarmkit这个dockerfile并没有在<https://github.com/rancher>下面的项目中，
这个需要顺藤摸瓜，找到该Dockerfile的维护者（其实也是Rancher的一名员工），
最终地址是<https://github.com/LLParse/swarmkit-catalog>。如图所示：  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1favahpg2pzj30kc0aiq57.jpg)  
swarmkit-mon中内置了docker，并映射了Host上的docker.sock，
这样可以在swarmkit-mon容器中控制docker创建swarmkit集群。swarmkit-mon的实现比较简单，
主要包括两个shell脚本：run.sh负责swarmkit集群的管理和Rancher的联动，
agent节点信息需要通过rancher-metadata读取，设置Host Label则直接调用Rancher API；
health.sh负责监控swarmkit节点的状态（通过与docker.sock通信读取Swarm.LocalNodeState的状态），
并与giddyup协作暴露健康检查端口，这样可以利用Rancher Cattle的healthcheck来保证swarmkit-mon服务的高可用性，
每个Host的swarmkit-mon出问题时可以进行自动重建恢复。原理如图：  
![](http://ww4.sinaimg.cn/large/006tKfTcjw1favaidbyfgj30bh08qmyh.jpg)

### 总结
目前来看，由于Kubernetes的发展的确迅猛，所以Rancher的更多精力都放在K8s上。
针对Swarmkit的支持显得略显单薄，但是Swarmkit本身的问题也很多，目前也难以应对复杂场景，
所以目前的支持力度应该是足够了。后续对docker1.13版本的Swarmkit支持也在持续迭代中。
