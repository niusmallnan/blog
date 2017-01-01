+++
categories = [
    "coding",
]
description = ""
date = "2017-01-01T07:42:28+08:00"
title = "Dead容器导致Rancher Server过载的解决方案"
slug = "rancher-server-overwhelmed-by-dead-containers"
menu = ""
banner = ""
images = [
]
tags = [
    "Rancher",
    "Docker"
]

+++
接之前的一篇文章[device or resource busy](http://niusmallnan.com/2016/12/27/docker-device-resource-busy/)，
其中描述了Docker的一个bug，这个bug其实会导致Rancher Server出现一个比较严重的问题。
<!--more-->
### 现象与原因
Rancher Server运行一段时间后，如若发现各种操作卡死，Host添加不上，Stack&Service也无法active，
且在proceeses列表里发现大量volumestoragepoolmap.remove任务，可以参考本文的解决方案：  
![](http://ww4.sinaimg.cn/large/006tKfTcjw1fbateimvjnj30iv0a40ug.jpg)

这些无法完成的volumestoragepoolmap.remove任务，背后其实就是那些dead containers，
分析之后，大致原因如下：

* docker本身的一个bug，导致一些容器删除失败，进入dead状态。
* Rancher的调度系统在不断重试删除这些dead容器，但是无法删除。
* 无法删除导致，这些删除任务不断加入rancher调度系统中，造成调度系统过载。
* 主机创建或是Stack/Service active请求都在排队，不能被迅速执行。

### 临时解决方案
依据之前的分析，processes里面出现了很多永远无法完成的任务，这些任务在不断的重试，这会压垮Rancher Server。
所以解决方案的目标就是，介入外部力量删除这些dead容器，升级docker engine是一个选择，
但是直接在生产环境升级是一个需要勇气的选择。比较温和的做法是定制脚本到每台agent节点上
强行删除这些dead容器。

考虑到这里有多台主机批量执行的场景，所以我们可以选择使用ansible工具，
创建一个工作目录如/opt/agent_upgrade，所有脚本都放到这个目录下。
首先我们要生成一个agent host的列表，这里可以借用Rancher的CLI工具，
具体脚本get_hosts.sh如下：
```
#!/bin/bash
#set -ex
rm -f all_hosts_ip

RANCHER="/usr/local/bin/rancher"

for env in `$RANCHER env -a --format {{.ID}}`; do
    for host in `$RANCHER --env $env hosts --format json| jq .Host.agentIpAddress| sed 's/\"//g'`; do
        echo "${host}" >> all_hosts_ip
    done
done
```

定义一个ansible的配置文件，由于所有agent host都是有运维专有访问秘钥的，
所以这里直接定义private_key_file比较方便，如下：
```
[defaults]
hostfile=/opt/agent_upgrade/all_hosts_ip
host_key_checking = False
remote_user = ubuntu
private_key_file=/opt/agent_upgrade/aws-prd.pem
```

然后我们就可以使用ansible来做批量操作了，删除dead容器的脚本非常简单，
clean_dead_containers.sh脚本如下：
```
#!/bin/bash

#set -ex
ansible all --sudo -m shell -a "docker ps -aq --filter status=dead > ~/dead" -vvvv
ansible all --sudo -m shell -a "cat ~/dead | xargs docker rm -f" -vvvv
```

由于Rancher集群在运行时仍然会产生这些dead containers，所以我们可以借用crontab来做定时清理，
添加一条crontab如下：
```
*/30 * * * * cd /opt/agent_upgrade && sh get_hosts.sh && sh clean_dead_containers.sh > /tmp/dead_containers.log 2>&1
```

如此，每隔30分钟定期清理这些dead containers，Rancher Server就不会受其影响。
