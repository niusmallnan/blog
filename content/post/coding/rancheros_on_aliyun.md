+++
description = ""
menu = ""
banner = ""
images = [
]
tags = [
    "RancherOS",
]
date = "2017-12-05T16:32:52+08:00"
title = "在阿里云上运行RancherOS"
slug = "rancheros-on-aliyun"
categories = [
    "coding",
]

+++
阿里云上也可以玩转RancherOS
<!--more-->
### 引言  
RancherLabs创立之初就立志于做一家纯粹的云产品公司，秉承一切开源的理念，为用户带来方便快捷的体验。我们推出的Rancher深受全球用户的喜爱，
为了构建完整的用户体验丰富我们的产品线，我们考虑把传统的操作系统也引入容器的体验，于是我们大概在2015年正式推出了[RancherOS](https://github.com/rancher/os)这个项目，
该项目一经推出凭借其极具创新性的理念，荣登2015年Github10大开源项目，我们致力于打造轻量可靠，一切皆容器的操作系统。今年早些时候Docker发布的Linuxkit其实也从侧面印证了，
我们选择的这条路是顺应发展趋势的，只不过RancherOS先行了一步。

现在我们在各个公有云平台上给用户免费使用，国外的厂商DigitalOcean甚至把RancherOS放到了Container distributions的UI选项中，越来越多的用户开始使用RancherOS，
参与到RancherOS的社区建设中，给我们反馈问题，协助我们不断改善产品。
而非常遗憾的是，中国用户始终无法在公有云中方便地使用它，其中原因诸多，国内的公有云也是这一两年才真正发展起来，
最初我们尝试在国内引入RancherOS时，很多平台无法提供有效的途径。

现在，一切已经开始走向成熟，这次可以很开心得向大家宣布，RancherOS登陆阿里云，欢迎大家使用。
作为开源软件，不断积累改进是我们的目标，文章最后给大家提供了各种反馈使用问题的渠道。

### 如何使用  
首先声明一下，作为支持阿里云的首个版本仍然有很多不成熟之处，还有一些使用限制，如下：

1. 需使用阿里云VPC网络
2. 创建VM时，只支持密钥，不支持设置密码

首先下载RancherOS专属阿里云镜像，比如：https://releases.rancher.com/os/latest/rancheros-aliyun.vhd

将镜像上传到阿里云对象存储中，因为自定义镜像是需要从对象存储中获取。

创建自定义镜像，按如下方式填写：  
![](https://ws4.sinaimg.cn/large/006tKfTcly1fmm2l7gfbhj312a0rewfl.jpg)
注意事项包括：系统盘大小大于10GB，系统架构为x86_64，系统平台为Others Linux，镜像格式为VHD。

创建VM时，按以下方式选择镜像：  
![](https://ws1.sinaimg.cn/large/006tKfTcly1fmwjd489e6j30wi09odg1.jpg)

创建成功后，使用ssh，访问用户是rancher，当然不要忘记指定密钥：  
![](https://ws2.sinaimg.cn/large/006tKfTcly1fm77ckj7ywj31ek0aq3z5.jpg)  
![](https://ws4.sinaimg.cn/large/006tKfTcly1fm76t2hvs7j316u12641q.jpg)

由于现在ROS镜像还没有发布到阿里云镜像市场中，当你创建完自定义镜像后，你可以按照如下方式共享给你的伙伴使用：  
![](https://ws1.sinaimg.cn/large/006tKfTcly1fm76xwulxxj30vy0jw74v.jpg)

### 反馈渠道  
各种问题欢迎随时提交issue:

- 这里建议使用英文 https://github.com/rancher/os/issues
- 中文issue可以提交到Maintainer的个人Repo上: https://github.com/niusmallnan/os/issues

同时也欢迎大家发送邮件(niusmallnan@gmail.com)给开发者。

