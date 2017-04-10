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
 |-mqtt_client.txt //mqtt支持，一种流行的IoT协议
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
 |-old  //不再使用，有一些嵌入式系统的例子
```

#### cc.h

此文件是编译器和处理器相关的头文件描述。

* Data types定义

u8_t，u16_t，u32_t，s8_t，s16_t，s32_t及mem_ptr_t

```
typedef unsigned char  u8_t;
```

* printf相关的数据定义

U16_F, S16_F, X16_F, U32_F, S32_F, X32_F, SZT_F一般被定义成 "hu", "d", "hx", "u", "d", "x", "uz"

```
#define U16_F  "hu"
```

* 大端小端定义

```
#define BYTE_ORDER  LITTLE_ENDIAN
或者
#define BYTE_ORDER  BIG_ENDIAN
```

TCP/IP协议栈采用大端模式，如果处理器也支持大端模式而且使用此模式，效率是最高的；但是大部分处理器使用小端模式，这就要注意使用htons()/htonl()/ntohs()/ntohl()函数进行转换。lwIP针对这种情况提供了标准的函数，但是如果处理器或编译器有相关的优化则应该封装成平台相关的函数替换使用。如下面这样的宏定义：

```
#define LWIP_PLATFORM_BYTESWAP  1
#define LWIP_PLATFORM_HTONS(x)  ((((u16_t)(x))>>8) | (((x)&0xFF)<<8))
#define LWIP_PLATFORM_HTONL(x)  ((((u32_t)(x))>>24) | (((x)&0xFF0000)>>8) | (((x)&0xFF00)<<8) | (((x)&0xFF)<<24))
```

* IP协议checksums选择

三种checksums算法选择：

```
1.load byte by byte, construct 16 bits word and add: not efficient for most platforms
2.load first byte if odd address, loop processing 16 bits words, add last byte.
3.load first byte and word if not 4 byte aligned, loop processing 32 bits words, add last word/byte.
```

定义下面的宏选择checksums：

```
#define LWIP_CHKSUM_ALGORITHM 2
```

如果是自定义checksums，则如下方式定义：

```
u16_t my_chksum(void *dataptr, u16_t len);
#define LWIP_CHKSUM  my_chksum
```

* 内存对齐

数据结构一般通过下面这种方式定义：

```
#ifdef PACK_STRUCT_USE_INCLUDES
#  include "arch/bpstruct.h"
#endif
PACK_STRUCT_BEGIN
struct <structure_name> {
  PACK_STRUCT_FIELD(<type> <field>);
  PACK_STRUCT_FIELD(<type> <field>);
  <...>
} PACK_STRUCT_STRUCT;
PACK_STRUCT_END
#ifdef PACK_STRUCT_USE_INCLUDES
#  include "arch/epstruct.h"
#endif
```

根据处理器和编译器的特点，需要定义几个宏，达到内存对齐，下面是以GCC为例的定义：

```
#define PACK_STRUCT_FIELD(x) x __attribute__((packed))
#define PACK_STRUCT_STRUCT __attribute__((packed))
#define PACK_STRUCT_BEGIN
#define PACK_STRUCT_END
```

* 平台相关的诊断输出定义

```
LWIP_PLATFORM_DIAG(x)  non-fatal，只是打印message
LWIP_PLATFORM_ASSERT(x)  fatal，打印message，然后停止执行
```

* 抢占保护



#### sys_arch.c

#### sys_arch.h

#### others
