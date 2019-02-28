+++
title = "K3s 初探"
description = ""
menu = ""
banner = ""
images = [
]
date = "2019-02-28T14:24:44+08:00"
slug = "k3s-init"
tags = [
    "K3S",
]
categories = [
    "coding",
]

+++
你等着 走着瞧！
<!--more-->

### 前言  
k3s发布仅仅几日，得到社区大量的反馈，大家的热情让我们感受到容器领域的创新依然还在路上。k3s顺利的成为了CNCF认证的kubernetes发行版，k3s的Github star数逼近3k，
容器技术圈广受尊重的大师Kelsey Hightower也发出推文表示赞赏。而Rancher所做的一切就是我们始终没有忘记初心，我们始终要做一家云基础设施厂商，并坚持创新，
Rancher/RKE/RancherOS/Longhorn/Rio/K3s这些产品不断的在扩展我们的技术版图。那么k3s到底是什么？主要创新点在哪里？让我们上手一试。

### 上手一试  
我们在AWS 启动两个虚拟机，系统RancherOS v1.5.1，当然你也可以换成你熟悉的OS，不过目前项目还处在发展阶段，有些OS的适配还没有完成，Ubuntu会是另一个不错的选择。
RancherOS是极致精简专为容器定制的Linux，使用RancherOS可以让我们体会到k3s对OS的依赖几乎很少。k3s已经使用了containerd替换Docker来做runtime，
所以我们可以在RancherOS停止Docker。containerd本身就是Docker的一部分，完全兼容我们所熟悉的Docker image。

```
system-docker stop docker

cd /opt && wget https://github.com/rancher/k3s/releases/download/v0.1.0/k3s
chmod +x k3s && ./k3s server &
```
默认server本身会自带agent，可以使用 --disable-agent 参数让其只提供server功能。

获取node token

```
cat /var/lib/rancher/k3s/server/node-token
```

添加额外的agent，node_token使用上面步骤返回的内容替换。同样我们已经不需要docker，依然在RancherOS中停止Docker。

```
system-docker stop docker
./k3s agent -s https://server-ip:6443 --token ${node_token} &
```

由于k3s移除了k8s中很多Legacy/alpha/non-default features，所以不要用一个特别复杂的yaml文 件来尝试。基本常用的deployment时支持的，所以我们可以部署一个deployment。

```
./k3s kubectl create -f https://kubernetes.io/docs/user-guide/nginx-deployment.yaml

./k3s kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           3m47s
```

当然我们也可以将k3s添加到Rancher中，目前支持导入方式，虽然还不是很完善，不过我们会持续不断更新提升体验。
你需要下载Rancher import集群时所需的yaml文件，与原生的k8s不同，这里k3s导入需要使用 k3s kubectl。

```
wget https://link-ip/v3/import/xx.yaml
./k3s kubectl apply -f xx.yaml
```

等待一段时间后，你可以看到k3s集群导入成功，由于k3s精简了很多k8s的组件，etcd默认是不启用的（默认使用sqlite3），所以有些报错暂时可以忽略。

![](https://ws1.sinaimg.cn/large/006tKfTcly1g0m4oedsjsj31p40u0ta6.jpg)


我们还可以用Rancher UI来创建简单的workload。

![](https://ws1.sinaimg.cn/large/006tKfTcly1g0m4pcel60j31dm0nmq3o.jpg)

### 一些思考  
Rancher在近些年越来越感受到outside datacenter的管理需求，不仅仅来自有工业物联网背景的制造业，甚至还有美国的快餐连锁行业，这些边缘计算的真实诉求推动着我们来创新。
将容器技术移植到边缘计算场景是个非常好的选择，容器拥有很好的生态系统，并且能够天然屏蔽硬件差异，带来部署管理上的极大便捷。
容器技术尤其是Kubernetes在数据中心层面愈发成熟，但是完全移植到边缘计算场景还是存在诸多问题，比如k8s对计算资源的消耗是边缘设备无法承受的，
同时很多k8s发行版无法支持ARM，而边缘设备目前是以ARM居多。这些其实就是Rancher创建k3s项目要解决的真实问题，k3s不仅仅是Rancher的产品，
我们还会推动它成为Kubernetes在边缘计算领域的标准。

未来Rancher容器管理平台会成为既可以管理datacenter k8s又可以管理outside k3s的产品，用户可以选择极致精简的容器操作系统RancherOS，
和专为容器而生的存储系统Longhorn来满足云内部和云之间的存储需求。在即将发布Rancher 2.2版本中，Rancher完成了对ARM的支持，
这样Rancher的纳管版图又扩大了很多，对边缘计算的支持将会更大。

### 延伸阅读  
1. K3s on rpi https://medium.com/@mabrams_46032/kubernetes-on-raspberry-pi-c246c72f362f
