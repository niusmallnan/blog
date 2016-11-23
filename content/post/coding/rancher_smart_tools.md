+++
date = "2016-08-15T14:16:42+08:00"
title = "扒一扒Rancher社区中的小工具"
slug = "rancher-smart-tools"
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
banner = ""

+++
除了我们所熟知的Rancher & RancherOS，Rancher Labs的开发团队在实践中提炼了很多实用的小工具，
这些小工具虽然并不会左右Rancher发展的大局，但是在项目标准化和开发效率上给团队带来巨大的便捷。
<!--more-->
### 引言
与Linux、OpenStack等成熟的技术社区相比，Rancher社区还是处于初级发展阶段，
一个技术社区的成败并不是单纯的代码贡献，而学习文档的数量和代码管理作业流程也是非常重要的。
如何让怀揣不同需求的工程师都能在社区中快速找到相应的解决方案，这就需要大家协同合作共同促进社区发展与完善。
除了我们所熟知的Rancher & RancherOS，Rancher Labs的开发团队在实践中提炼了很多实用的小工具，
这些小工具虽然并不会左右Rancher发展的大局，但是在项目标准化和开发效率上给团队带来巨大的便捷。
这次主要是带着大家一起来认识一下这些小工具。

##### Golang包管理工具-Trash
项目地址：<https://github.com/rancher/trash>  
目前主流的编程语言 Python、Ruby、Java、Php 等已经把包管理的流程设计的犹如行云流水般流畅，
一般情况下开发者是不需要操心类库包依赖管理以及升级、备份、团队协作的。Golang在1.5版本开始，
官方开始引入包管理的设计，加了 vendor目录来支持本地包管理依赖，
但是需要特殊设置 GO15VENDOREXPERIMENT=1，在1.6时代这个特性已经是默认的了。
可是vendor并没有统一的版本号管理功能，只是额外提供了project内包的依赖路径。
于是Trash这个工具就应运而生了，Trash的使用非常简单，只需要有一份依赖包的描述文件即可。

描述文件 trash.conf 支持两种格式，普通方式和YAML方式，
可以直接在其中描述依赖库的远程地址、版本号等，一个简单的例子（我这里使用普通格式）：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa20qdpd40j20c302t0t0.jpg)

然后在根目录执行trash，即可获得相关版本的依赖包，非常轻量级，非常简洁：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa20qtj6kcj20av06cjrr.jpg)

##### Golang编译工具-Dapper
项目地址：<https://github.com/rancher/dapper>  
我们在编译golang执行程序的时候，因为涉及到协作开发，这就会碰到一个问题，
就是如何保证编译环境的一致性。环境一致性最好的办法就是使用容器技术，
Dapper就是利用Docker build镜像的过程中可以执行各种命令生成容器的原理。
只需在项目的根目录下创建 Dockerfile.dapper 文件，这是一个参考Dockerfile标准的文件，
执行dapper命令即可按照约定规则生成最终的执行程序，通过这种方式统一的编译环境。

几乎所有的Rancher项目都是基于Dapper来编译的，随意打开一个项目比如
[rancher-dns](https://github.com/rancher/rancher-dns)就可以看到Dockerfile.dapper文件：  
![](http://ww3.sinaimg.cn/large/7853084cjw1fa20thy5rhj20j809ojtr.jpg)

DAPPER_SOURCE 指定了容器内的源码路径  
DAPPER_OUTPUT 指定了编译后的输出路径，bin dist 目录会自动在项目根目录下创建  
DAPPER_DOCKER_SOCKET 设置为True相当于`docker run -v /var/run/docker.sock:/var/run/docker.sock ...`  
DAPPER_ENV 相当于`docker run -e TAG -e REPO ...`

有一点需要注意的是，目前Dapper装载源码有两种方式bind和cp，bind就是直接mount本地的源码路径，
但是如果使用remote docker daemon方式那就得使用cp模式了。

##### Golang项目标准化工具 Go-skel
项目地址：<https://github.com/rancher/go-skel>  
介绍了包管理工具和打包编译工具，如果我们在创建一个golang项目时能把这两个工具整合起来一起使用，
那就太赞了。go-skel就是提供了这样一个便利，我们直接来demo一下。

clone一份go-skel的源码，创建一个rancher-tour（./skel.sh rancher-tour）的项目：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa20wzh1pxj20jt09v0vf.jpg)

执行完毕后，会创建一个标准的项目，包含了dapper和trash这两个工具，
同时定义了一份Makefile，我们可以通过make命令来简化操作：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa20xj5jd3j20cw0art9r.jpg)

比如我们要执行一个ci操作，那就可以直接运行 make ci，自动运行单元测试，
并在bin目录下生成最终的可执行程序：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa20xw28tzj20gz05kmyl.jpg)

标准项目已经创建了一些初始化代码，集成了 github.com/urfave/cli ，所以我们可以执行执行rancher-tour：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa20y6wzdjj20kq08xgmw.jpg)

##### 微服务辅助小工具 Giddyup
项目地址：<https://github.com/cloudnautique/giddyup>  
一个传统的服务在容器化的过程中，通常我们会将其拆分成多个微服务，
充分展现每个容器只做好一件事情这样的哲学。那么就会出现我们在Rancher中看到的sidekick容器，
数据卷容器，专门负责更新配置信息的容器等等。实际应用中我们会遇到一些问题，
比如这些微服务容器的启动是无序的，无序启动会导致微服务之间可能出现连接失败，
进而导致整个服务不可用；再比如我们如何来判定依赖的微服务已经正常启动，
这可能需要一个health check的服务端口。giddyup就是简化这种微服务检查工作的利器，它都能做些什么呢：

1. Get connection strings from DNS or Rancher Metadata.
2. Determine if your container is the leader in the service.
3. Proxy traffic to the leader.
4. Wait for service to have the desired scale.
5. Get the scale of the service.
6. Get Managed-IP of the container (/self/container/primary_ip).

我们还是创建stack并在其中创建service来体验一下giddyup的部分功能，
service中包含两个容器分别为主容器main和sidekick容器conf，他们共享网络栈：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa210w5l9pj20c807xt9a.jpg)

我们可以在conf内启动giddyup health，会启动一个监听1620端口的http服务，服务路径是/ping：  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa2115foy9j20a703h3yl.jpg)

此时我们可以在其他服务的容器内查看这个服务 health port的健康状态，
通过giddyup ip stringify获取服务地址，使用giddyup probe可以查看相关状态：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa211iwkz2j20ql03jgmj.jpg)

可以看到probe返回了OK的信息，说明giddy/main的health port是正常的，
如果我们把giddyup health的端口停掉，giddyup probe会如何？  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa211ulol7j20mw06kn1u.jpg)

除了开启health port功能外，还可以sleep等待service内所有容器都启动，
比如刚才的service，我在main上做wait操作，同时在UI上对service扩容，
可以看到为了等待完成扩容差不多sleep了3s：  
![](http://ww1.sinaimg.cn/large/7853084cjw1fa212n0tykj20ep031mxg.jpg)

更多功能可以去看一看giddyup项目的README文档，另外也可以看看catalog中的一些项目是怎么使用giddyup的，
参考：<https://github.com/rancher/catalog-dockerfiles>

##### Rancher CLI 工具
项目地址：<https://github.com/rancher/cli>  
现在我们管理Rancher的方式包括UI、API方式，为了能够和其他工具更好的融合Rancher开发了CLI工具，
可以非常轻量级的嵌入到其他工具中。CLI将会在Rancher 1.2-pre2版本中正式放出，
兼容性上目前来看没有支持老版本的计划，所以在老版本上出现各种问题也是正常的。

直接在<https://github.com/rancher/cli/releases>下载最新版本即可试用，
将rancher cli放到PATH中，正式使用前需要初始化：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa214oqzn5j20hy07a405.jpg)

然后便可以执行各种犀利的操作，比如针对某个service进行scale操作，
这比之前需要用rancher-compose cli要简洁的多：  
![](http://ww2.sinaimg.cn/large/7853084cjw1fa2155xxeaj20ps05ggng.jpg)

再比如可以实时监听rancher events，可以即时查看Rancher中正在发生的各种任务，
对协助排查问题，分析系统运行情况都非常有用：  
![](http://ww4.sinaimg.cn/large/7853084cjw1fa215gzjcrj20dr077myp.jpg)

#### 后语
小工具中内含着大智慧，工具文化是工程师Team高效协作的重要标志，
高质量的工具能让项目开发有事半功倍之效，其背后也蕴藏着深厚的团队文化理念，
就是不计项目KPI利用个人业余时间为团队做贡献的和谐氛围。
其实国内很多互联网公司都是有专门设立工具开发工程师的岗位，
对工具带来的生产效率提升，其重视程度不言而喻！


