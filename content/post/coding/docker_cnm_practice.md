+++
date = "2016-09-19T17:22:00+08:00"
title = "CNM macvlan实践"
slug = "docker-cnm-practice"
banner = ""
images = [
]
tags = [
    "Docker",
]
categories = [
    "coding",
]
description = ""
menu = ""

+++
针对Dcoker原生网络模型CNM的一次实践记录，网络模式macvlan。

<!--more-->
首先测试一下系统对bridge和namespace的支持情况：
```
# add the namespaces
ip netns add ns1
ip netns add ns2

# create the macvlan link attaching it to the parent host eno1
ip link add mv1 link eno1 type macvlan mode bridge
ip link add mv2 link eno1 type macvlan mode bridge

# move the new interface mv1/mv2 to the new namespace
ip link set mv1 netns ns1
ip link set mv2 netns ns2

# bring the two interfaces up
ip netns exec ns1 ip link set dev mv1 up
ip netns exec ns2 ip link set dev mv2 up

# set ip addresses
ip netns exec ns1 ifconfig mv1 192.168.1.50/24 up
ip netns exec ns2 ifconfig mv2 192.168.1.60/24 up

# show interface detail
ip netns exec ns1 ip a
ip netns exec ns2 ip a

# ping from one ns to another
ip netns exec ns1 ping -c 4 192.168.1.60

# cleanup
ip netns del ns1
ip netns del ns2
```

给网卡创建一个vlan设备：
```
# create a new subinterface tied to dot1q vlan 1045
ip link add link eno1 name eno1.1045 type vlan id 1045

# assign an IP addr
ip a add 192.168.120.24/24 dev eno1.1045

# enable the new sub-interface
ip link set eno1.1045 up

# remove sub-interface
#ip link del eno1.1045
```

创建docker network：
```
docker network create -d macvlan \
    --subnet=192.168.225.0/24 \
    --gateway=192.168.225.1 \
    -o parent=eno1.1045 macvlan1045
```

创建容器，测试网络连通性：
`docker run --net=macvlan1045 --rm --ip=192.168.225.24 -it alpine /bin/sh`


