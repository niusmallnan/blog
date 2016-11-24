+++
description = ""
date = "2016-09-23T17:12:27+08:00"
title = "Rancher HA部署实践"
slug = "rancher-ha-practice"
menu = ""
banner = ""
images = [
]
tags = [
    "Rancher",
]
categories = [
    "coding"
]
+++
Rancher1.1版本HA结构部署实践记录
<!--more-->
### 过程记录
以三节点为例，节点信息：
ubuntu(aufs)+docker1.11.2+rancher1.1.x
Haproxy+Mysql 放在同一节点 使用VM
Rancher HA Node 三个节点 使用baremetal（也可以使用VM，建议使用4核8G以上flavor）

先部署Haproxy：
```
docker run -d --name rancher-haproxy \
           -v /opt/conf/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro \
           -p 80:80 \
           haproxy
```

文件 /opt/conf/haproxy.cfg参考：
```
global
        log 127.0.0.1 local0
        log 127.0.0.1 local1 notice
        maxconn 4096
        maxpipes 1024
        daemon

defaults
        log     global
        mode    tcp
        option  tcplog
        option  dontlognull
        option  redispatch
        option http-server-close
        option forwardfor
        retries 3
        timeout connect 5000
        timeout client 50000
        timeout server 50000

frontend default_frontend
        bind *:80
        mode http

        default_backend rancher-ha-node

backend rancher-ha-node
        mode http
        server r-ha-1 xx.xx.xx.xx
        server r-ha-2 xx.xx.xx.xx
        server r-ha-3 xx.xx.xx.xx
```

创建容器化的mysql：
```
docker run --name rancher-mysql \
    -v /opt/data/mysql:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=root \
    -p 3306:3306 \
    -d mysql:5.5

mysql -uroot -proot
CREATE DATABASE IF NOT EXISTS cattle COLLATE = 'utf8_general_ci' CHARACTER SET = 'utf8';
GRANT ALL ON cattle.* TO 'cattle'@'%' IDENTIFIED BY 'cattle';
GRANT ALL ON cattle.* TO 'cattle'@'localhost' IDENTIFIED BY 'cattle';
```

正常运行rancher，配置并下载HA脚本，如果不想用HTTPS，
记得把**CATTLE_HA_HOST_REGISTRATION_URL**的值换成HTTP的，再执行脚本。

