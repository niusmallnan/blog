+++
description = ""
menu = ""
banner = ""
images = [
]
tags = [
    "RancherOS",
]
categories = [
    "coding",
]
date = "2018-01-22T17:16:17+08:00"
title = "如何在RancherOS上更新microcode"
slug = "update-microcode-on-rancheros"

+++
RancherOS上更新microcode
<!--more-->
### 引言
进来公布Google的Project Zero团队公布的Spectre和Meltdown漏洞，真可谓一石激起千层浪，
业内各个硬件软件厂商开源社区都在积极参与漏洞修复。Meltdown是最早被修复的，在Kernel中开启KPTI就可以缓解该问题，
Spectre则依然在火热进行中。而作为漏洞的始作俑者Intel，Spectre漏洞的Intel官方缓解方案也于近日放出。
当然目前很多更新并非真正的修复，而是在最大程度抵御和缓解相关攻击。

### 更新microcode
通过更新microcode和系统补丁可以缓解Spectre Var. 2漏洞，也就是CVE-2017-5715 branch target injection。
除了在BIOS中可以更新microcode之外，Kernel也提供了更新microcode的机制。本文只关注在如何在RancherOS上更新microcode。

下载最新的Intel microcode版本，注意microcode一般并不是适用所有的CPU，比如[20180108](https://downloadcenter.intel.com/download/27431/Linux-Processor-Microcode-Data-File
)这个版本，主要是以下CPU才会起作用：  
![](https://ws1.sinaimg.cn/large/006tNc79ly1fnpsvdzc3rj30wu1byn1l.jpg)

下载并解压缩之后，你会发现一个是intel-ucode目录下一些文件，另外一个单独的microcode.dat文件。
前者是支持热加载方式，也是现在比较推荐的方式；后者是传统更新方式，需要在initrd中加入microcode加载。
对于RancherOS，前者比较合适，使用起来相对简单。

若要在内核支持更新microcode，需要在编译内核时加入以下配置（RancherOS已经开启）：
```
CONFIG_MICROCODE=y
CONFIG_MICROCODE_INTEL=y
CONFIG_MICROCODE_AMD=y
CONFIG_MICROCODE_OLD_INTERFACE=y #支持传统方式，开启此项
```

安装过程比较简单，几乎每个版本的microcode都有相应的releasnote，大致如下：
```
# Make sure /sys/devices/system/cpu/microcode/reload exits:
$ ls -l /sys/devices/system/cpu/microcode/reload

# You must copy all files from intel-ucode to /lib/firmware/intel-ucode/ using the cp command
$ sudo cp -v intel-ucode/* /lib/firmware/intel-ucode/

# You just copied intel-ucode directory to /lib/firmware/. Write the reload interface to 1 to reload the microcode files:
$ echo 1 > /sys/devices/system/cpu/microcode/reload
```

无论更新成功与否，在dmesg中都会查看到相关信息，比如：
```
$ dmesg | grep microcode
[   13.659429] microcode: sig=0x306f2, pf=0x1, revision=0x36
[   13.665981] microcode: Microcode Update Driver: v2.01 <tigran@aivazian.fsnet.co.uk>, Peter Oruba
[  510.899733] microcode: updated to revision 0x3b, date = 2017-11-17
```

查看cpuinfo，再次确认microcode版本，不同的CPU型号，升级后对应的microcode版本是不同的：
```
$ cat /proc/cpuinfo |grep "model\|microcode\|stepping\|family" |head -n 5
cpu family   : 6
model        : 63
model name   : Intel(R) Xeon(R) CPU E5-2640 v3 @ 2.60GHz
stepping     : 2
microcode    : 0x3b #之前是0x36
```

然后在cloud-config中在runcmd添加脚本，保证每次启动都加载最新版本的microcode：
```
runcmd:
- echo 1 > /sys/devices/system/cpu/microcode/reload
```

### 总结
关于Spectre Var. 2，我们依然在持续关注，
直接在内核编译中使用Retpoline指令替换技术，可以更简单方便的缓解branch target injection，
现在内核已经支持了Retpoline指令替换的设置，但是也需要最新版本的GCC编译器的特性支持，
而带有GCC的新补丁的正式版本还没有发布，一旦GCC新版本发布，我们会马上更新内核并发布新的RancherOS版本。
