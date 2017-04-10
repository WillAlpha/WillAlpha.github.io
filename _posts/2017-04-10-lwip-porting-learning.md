---
layout: post
title: "lwIP学习笔记-移植"
categories: RTOS
tags: Embedded TCP/IP lwIP
author: Will
---

* content
{:toc}


[lwIP](http://savannah.nongnu.org/projects/lwip/)是瑞典计算机科学院开源(BSD license)的轻量级TCP/IP协议栈，主要应用于嵌入式系统，在保持TCP/IP协议的主要功能的基础上，减少内存使用，优化处理流程，减少代码量等等。之前大部分使用的是v1.4.x版本，差不多4~5年后直接更新到了v2.x.x版本。

## 源码

lwIP的源码可以到官网的Source Code页面[下载](http://git.savannah.nongnu.org/cgit/lwip.git)。下载v2.0.1 release版本，源码目录：

```
doc/  //文档
 |-contrib.txt
 |-mdns.txt
 |-mqtt_client.txt
 |-ppp.txt
 |-rwaapi.txt
 |-savannah.txt
 |-sys_arch.txt
 |-...
src/  //源码
 |-api  //high-level api，如果直接使用low-level api则用不到
 |-apps  //应用代码，调用low-level raw api
 |-core  //核心代码，TPC/IP stack/protocol implementations/
 |       //memory and buffer management/the low-level raw api
 |-include  //头文件
 |-netif  //network interface device drivers
 |-...
test/ //测试
 |-fuzz  //新增linux环境下fuzzing测试，使用‘american fuzzy lop’工具
 |-unit //单元测试，同v1.4.x之前的版本
README
...
```

另外，contrib的包也有用，找到对应的版本的contrib包[下载](http://git.savannah.gnu.org/cgit/lwip/lwip-contrib.git)。后面用到时再说明，下面是contrib包里的内容：

```
addons/
apps/
Coverity/
ports/
```

## 移植

官方给出了[Platform Developers Manual](http://lwip.wikia.com/wiki/LwIP_Platform_Developers_Manual)，与移植相关的部分：

```
Creating a platform
  -Porting for an OS (sys_arch.c/h, cc.h)
  -Porting for an OS 1.4.0 (sys_arch.c/h, cc.h)
  -Porting for Bare Metal environment (no OS)
  -Writing a device driver (netif,ethernetif...)
  -Available device drivers written by lwIP users
```

lwIP可以移植到基于OS平台或者无OS的平台上，一般还是在OS的平台上使用居多，因此直接看基于OS的移植，关注v2.x.x且基于OS的[移植文档](http://lwip.wikia.com/wiki/Porting_for_an_OS)。

### Porting for an OS

开发者需要实现几个文件来适配OS，如cc.h/sys_arch.c/sys_arch.h；源码里面的doc/sys_arch.txt可以先阅读以下，是sys_arch相关的说明。上面提到的相应版本的contrib的包里面的ports/目录下的内容也非常值得借鉴，已经针对不同的平台实现了移植相关的文件。

```
ports/
 |-unix  //类unix系统的移植
 |-win32  //windows系统的移植
 |-old  //不再使用
```

* cc.h



