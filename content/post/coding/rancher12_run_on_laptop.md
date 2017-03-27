+++
images = [
]
date = "2016-12-02T21:31:53+08:00"
title = "实践指南-快速解锁Rancher v1.2"
slug = "rancher12-run-on-laptop"
tags = [
    "Rancher",
    "Rancher v1.2"
]
categories = [
    "coding",
]
description = ""
menu = ""
banner = ""

+++
Rancher v1.2系列文章，已独家授权Rancher Labs，转载请联系Rancher China相关人员。
<!--more-->
### 引言
Rancher v1.2已经发布，相信众多容器江湖的伙伴们正魔拳擦准备好好体验一番。
由于Docker能够落地的操作系统众多，各种Docker版本不同的Graph driver，
所以通常大版本的第一个release都会在兼容性上有一些小问题。
为了更好的体验Rancher v1.2的完整特性，我们选取了Rancher测试比较严格的运行环境。
手握众多服务器资源的devops们可以飘过此文，身背MBP或Windows笔记本的Sales/Pre-Sales们可以品读一番。

### 基础软件安装
首先需要安装基础软件，由于Rancher v1.2已经支持Docker v1.2，
所以可以直接使用Docker的Mac或Windows版（以下以Mac为例），
下载地址：<https://www.docker.com/>。在Mac上，
Docker会使用xhyve轻量级虚拟化来保证一个Linux环境，所以可以把Rancher Server直接运行起来。

因为要在MBP上添加多个Host组成小集群，所以需要用虚拟化扩展多个节点添加到Rancher集群中。
这里可以使用docker-machine控制VirtualBox来添加节点，
VirtualBox下载地址：<https://www.virtualbox.org/wiki/Downloads>。

在Host节点的操作系统上，可以选取RancherOS，我们的目标是快速体验新特性，
而Rancher Labs在Rancher和RancherOS的相互兼容性上是做了大量测试的，
这样可以避免我们少进坑，直接体验新特性。
RancherOS下载地址：<https://github.com/rancher/os>，推荐使用最新release版本。

在用docker-machine驱动VirtualBox来创建Host时，可以指定操作系统ISO的URL路径，
由于我们使用RancherOS，所以最好把RancherOS放到本机HTTP服务器内。
MBP内自带Apache HTTPD，将Apache的vhosts模块开启，并添加配置：
```
# 开启vhost /etc/apache2/httpd.conf
# 以下两行的默认注释去掉
LoadModule vhost_alias_module libexec/apache2/mod_vhost_alias.so
Include /private/etc/apache2/extra/httpd-vhosts.conf

# vhost的配置 /etc/apache2/extra/httpd-vhosts.conf
# DocumentRoot目录就是在用户根目录下创建Sites
# 如用户名niusmallnan，则DocumentRoot就是/Users/niusmallnan/Sites
<VirtualHost *:80>
    DocumentRoot "/Users/niusmallnan/Sites"
    ServerName localhost
    ErrorLog "/private/var/log/apache2/sites-error_log"
    CustomLog "/private/var/log/apache2/sites-access_log" common
    <Directory />
        Options Indexes FollowSymLinks MultiViews
        AllowOverride None
        Order allow,deny
        Allow from all
        Require all granted
    </Directory>
</VirtualHost>

# 重启 Apache
$ sudo apachectl restart

# 拷贝 RancherOS的ISO 到 DocumentRoot
$ cp rancheros.iso /Users/niusmallnan/Sites/
```

### Rancher安装
首先打开Docker，并配置registry mirror，配置完成后重启Docker。
mirror的服务可以去各个公用云厂商申请一个，比如我这里使用的是阿里云的registry mirror，
如图所示：  
![](http://ww3.sinaimg.cn/large/006tKfTcjw1fav9vw30d1j30jh082403.jpg)

打开terminal，安装Rancher Server：  
`$ docker run -d --restart=unless-stopped -p 8080:8080 rancher/server:stable`

若要添加Host节点，则需要通过docker-machine创建Host，
这里使用的规格是2核2G（具体可根据自身MBP的性能调整），脚本（add_ros_host.sh）参考如下：
```
#!/usr/bin/env bash
ROS_ISO_URL='http://127.0.0.1/rancheros.iso'
ROS_CPU_COUNT=2
ROS_MEMORY=2048
docker-machine create -d virtualbox \
        --virtualbox-boot2docker-url $ROS_ISO_URL \
        --virtualbox-cpu-count $ROS_CPU_COUNT \
        --virtualbox-memory $ROS_MEMORY \
        $1
docker-machine ls
```
添加节点则需执行：  
`$ ./add_ros_host.sh ros-1`  
添加完成后，可以进入虚机内进行设置：
```
$ docker-machine ls
NAME  ACTIVE DRIVER     STATE     URL                       SWARM DOCKER ERRORS 
ros-1 -      virtualbox Running   tcp://192.168.99.100:2376       v1.12.3

# 进入VM中
$ docker-machine ssh ros-1
# RancherOS内设置registry mirror
$ sudo ros config set rancher.docker.extra_args \
        "['--registry-mirror','https://s06nkgus.mirror.aliyuncs.com']"
$ sudo system-docker restart docker
```

由于我们要使用VirtualBox的虚机组成一个小集群，所以建议把Rancher的Host Registration URL
设置为<http://192.168.99.1:8080>，如下图所示：  
![](http://ww4.sinaimg.cn/large/006tKfTcjw1fav9z6dih7j30eo072gmf.jpg)  
添加Rancher agent的时候也要注意，CATTLE_AGENT_IP参数要设置成虚机内192.168.99.0/24网段的IP，
如下图所示：  
![](http://ww1.sinaimg.cn/large/006tKfTcjw1fav9zo3g5zj30hp06tmyk.jpg)  
如此就可以基本完全解锁Rancher v1.2的各种功能了，完整演示各种特性。

### 总结
Docker目前版本分支众多，虽然最新的v1.13即将发布，但是各个公司的使用版本应该说涵盖了v1.9到v1.12，
而且Docker graph driver也有很多，再加上很多的LinuxOS，可以说使用Docker而产生组合有很多种，
这就会带来各种各样的兼容性问题，因此导致的生产环境故障会让人头疼不已。
当然如果纯粹基于演示和调研新功能，我们可以优先兼容性较好的选择。
