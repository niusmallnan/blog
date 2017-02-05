+++
title = "Rancher K8s Helm使用体验"
slug = "rancher-k8s-helm-practice"
tags = [
    "Rancher",
    "Kubernetes",
    "Helm",
]
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""
images = [
]
date = "2017-02-05T14:08:38+08:00"

+++
潇潇洒洒玩一玩
<!--more-->
### 引言
之前的文章已经阐述了Rancher K8s中国区的使用优化，
<http://niusmallnan.com/2017/01/19/optimize-rancher-k8s-in-china/>，
本文则是选取一个特殊的组件Helm，深入一些体验使用过程。Helm是K8s中类似管理catalog的组件，
在较新的Rancher K8s版本中，Helm已经默认集成可直接使用。

### 初始化CLI环境
如果已经习惯使用Rancher UI上自带的kubectl tab，那么可以跳过此步。
大部分玩家还是更喜欢在自己的机器上使用kubectl和helm CLI来做管理的。
在自己的机器上部署kubectl和helm命令行工具，有一个比较偷懒的方法，
就是直接到kubectld容器拷贝出来，主要过程如下：
```
# kubectrld容器ID为 42291346c064
$ docker cp 42291346c064:/usr/bin/kubectl /usr/local/bin/kubectl
$ docker cp 42291346c064:/usr/bin/helm /usr/local/bin/helm
$ docker cp 42291346c064:/tmp/.helm ~/.helm
```

当然也不要忘记kubectl的配置文件，在Rancher UI上生成，然后拷贝到对应的本地目录上。  
![](https://ww1.sinaimg.cn/large/006tNc79ly1fcfkc0a5yjj30dx07edgb.jpg)

### 基于Helm部署Mysql
K8s官方的Charts中已经有了mysql这个应用，
这里<https://github.com/kubernetes/charts/tree/master/stable>可以找到，
几乎所有的Chart都需要依赖PersistentVolumeClaim提供卷，所以在一切开始之前，
我们需要先创建一个PersistentVolume，来提供一个数据卷池。这里可以选择部署比较方便的NFS。

选择一台主机，安装nfs-kernel-server，并做相应配置：
```
$ apt-get install nfs-kernel-server

# 配置卷
# 修改 /etc/exports，添加如下内容
/nfsexports *(rw,async,no_root_squash,no_subtree_check)

# 重启nfs-server
service nfs-kernel-server restart
```

使用kubectl创建PV，文件内容如下：
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
  annotations:
    volume.beta.kubernetes.io/storage-class: "slow"
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /nfsexports #NFS mount path
    server: 172.31.17.169 #NFS Server IP
```

安装mysql之前，需要准备一份mysql chart的配置文件，也就是对应的values.yaml，
其他内容<https://github.com/kubernetes/charts/blob/master/stable/mysql/values.yaml>基本不变，
主要修改persistence部分，这样所依赖的PVC才能bound到对应的PV上，如下：
```
persistence:
  enabled: true
  storageClass: slow
  accessMode: ReadWriteMany
  size: 1Gi
```

一切准备妥当，就可以进行Mysql Chart的安装，执行过程如下：`helm install --name ddb -f values.yaml stable/mysql`。

安装完毕后，在Rancher UI和K8s Dashboard上都可以看到。  
![](https://ww1.sinaimg.cn/large/006tNc79ly1fcfkfju2ayj30k407q750.jpg)  
![](https://ww3.sinaimg.cn/large/006tNc79ly1fcfkfsuoayj30ed05zaae.jpg)

