---
layout: post
title: "lwIP学习笔记-移植"
categories: TCP/IP
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

```c
typedef unsigned char  u8_t;
```

* printf相关的数据定义

U16_F, S16_F, X16_F, U32_F, S32_F, X32_F, SZT_F一般被定义成 "hu", "d", "hx", "u", "d", "x", "uz"

```c
#define U16_F  "hu"
```

* 大端小端定义

```c
#define BYTE_ORDER  LITTLE_ENDIAN
或者
#define BYTE_ORDER  BIG_ENDIAN
```

TCP/IP协议栈采用大端模式，如果处理器也支持大端模式而且使用此模式，效率是最高的；但是大部分处理器使用小端模式，这就要注意使用htons()/htonl()/ntohs()/ntohl()函数进行转换。lwIP针对这种情况提供了标准的函数，但是如果处理器或编译器有相关的优化则应该封装成平台相关的函数替换使用。如下面这样的宏定义：

```c
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

```c
#define LWIP_CHKSUM_ALGORITHM 2
```

如果是自定义checksums，则如下方式定义：

```c
u16_t my_chksum(void *dataptr, u16_t len);
#define LWIP_CHKSUM  my_chksum
```

* 内存对齐

数据结构一般通过下面这种方式定义：

```c
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

```c
#define PACK_STRUCT_FIELD(x) x __attribute__((packed))
#define PACK_STRUCT_STRUCT __attribute__((packed))
#define PACK_STRUCT_BEGIN
#define PACK_STRUCT_END
```

* 平台相关的诊断输出定义

```c
LWIP_PLATFORM_DIAG(x)  non-fatal，只是打印message
LWIP_PLATFORM_ASSERT(x)  fatal，打印message，然后停止执行
```

* 抢占保护

类似FreeRTOS里面的`taskENTER_CRITICAL()`和`taskEXIT_CRITICAL()`，lwIP源码里面使用如下面大写的宏定义，默认在src/include/lwip/sys.h里定义，lwIP推荐不要在cc.h里面定义这些宏，而是在lwipopts.h里将宏SYS_LIGHTWEIGHT_PROT=1，然后sys.h里的默认定义会生效，如下所示；然后在sys_arch.h和sys_arch.c具体实现。

```c
#define SYS_ARCH_DECL_PROTECT(lev) sys_prot_t lev
#define SYS_ARCH_PROTECT(lev) lev = sys_arch_protect()
#define SYS_ARCH_UNPROTECT(lev) sys_arch_unprotect(lev)
```

#### sys_arch.c

此文件需要实现semaphores和mailboxes给lwIP使用。

* Semaphores

lwIP使用Counting Semaphores和Binary Semaphores。Semaphores的数据结构定义在sys_arch.h里面，源码并未给出具体的数据结构定义，完全交给开发者根据自己使用的RTOS自行决定，数据结构定义为sys_sem_t，而且需要实现下面这些函数

```c
sys_sem_t sys_sem_new(u8_t count);
void sys_sem_free(sys_sem_t sem);
void sys_sem_signal(sys_sem_t sem);
u32_t sys_arch_sem_wait(sys_sem_t sem, u32_t timeout);
```

* Mailboxes

与Semaphores类似，需要根据具体的RTOS实现数据结构sys_mbox_t，像FreeRTOS是没有Mailboxes的，但是可以使用Queue代替。需要实现下面的这些函数

```c
sys_mbox_t sys_mbox_new(int size);
void sys_mbox_free(sys_mbox_t mbox);
void sys_mbox_post(sys_mbox_t mbox, void *msg);
u32_t sys_arch_mbox_fetch(sys_mbox_t mbox, void **msg, u32_t timeout);
u32_t sys_arch_mbox_tryfetch(sys_mbox_t mbox, void **msg);
err_t sys_mbox_trypost(sys_mbox_t mbox, void *msg);
```

* Timeouts/Threads

RTOS一般都提供，也可以再做一次封装。

* System

实现lwIP的系统初始化函数`sys_init(void)`

#### sys_arch.h

需要定义下面的数据结构，结合具体使用的RTOS进行封装即可

```c
sys_sem_t
sys_mbox_t
sys_thread_t
```

SYS_MBOX_NULL, SYS_SEM_NULL直接定义成NULL。还有就是之前提到过的抢占保护，sys_arch.h中声明，sys_arch.c实现。

```c
sys_prot_t sys_arch_protect(void);
void sys_arch_unprotect(sys_prot_t pval);
```

#### memory management

lwIP默认使用自己的堆管理，具体实现在mem.c/memp.c里面，需要特别注意使用。


### Writing a device driver

netif/ethernetif.c文件默认是注释掉的，这是一个很好的网卡驱动模板。需要完善和修改三个函数

```c
struct ethernetif {
  struct eth_addr *ethaddr;
  /* Add whatever per-interface state that is needed here. */
  //在这里可以添加自己的私有数据结构，不是必须的
};

static void low_level_init(struct netif *netif);
static err_t low_level_output(struct netif *netif, struct pbuf *p);
static struct pbuf *low_level_input(struct netif *netif);

static void low_level_init(struct netif *netif)
{
  struct ethernetif *ethernetif = netif->state;

  /* set MAC hardware address length */
  netif->hwaddr_len = ETHARP_HWADDR_LEN;

  //添加网卡的mac地址
  /* set MAC hardware address */
  netif->hwaddr[0] = ;
  netif->hwaddr[1] = ;
  netif->hwaddr[2] = ;
  netif->hwaddr[3] = ;
  netif->hwaddr[4] = ;
  netif->hwaddr[5] = ;

  /* maximum transfer unit */
  netif->mtu = 1500;

  /* device capabilities */
  /* don't set NETIF_FLAG_ETHARP if this device is not an ethernet one */
  netif->flags = NETIF_FLAG_BROADCAST | NETIF_FLAG_ETHARP | NETIF_FLAG_LINK_UP;

#if LWIP_IPV6 && LWIP_IPV6_MLD
  /*
   * For hardware/netifs that implement MAC filtering.
   * All-nodes link-local is handled by default, so we must let the hardware know
   * to allow multicast packets in.
   * Should set mld_mac_filter previously. */
  if (netif->mld_mac_filter != NULL) {
    ip6_addr_t ip6_allnodes_ll;
    ip6_addr_set_allnodes_linklocal(&ip6_allnodes_ll);
    netif->mld_mac_filter(netif, &ip6_allnodes_ll, NETIF_ADD_MAC_FILTER);
  }
#endif /* LWIP_IPV6 && LWIP_IPV6_MLD */

  //其他相关的初始化
  /* Do whatever else is needed to initialize interface. */
}

static err_t low_level_output(struct netif *netif, struct pbuf *p)
{
  struct ethernetif *ethernetif = netif->state;
  struct pbuf *q;

  //实现初始化transfer()
  initiate transfer();

#if ETH_PAD_SIZE
  pbuf_header(p, -ETH_PAD_SIZE); /* drop the padding word */
#endif

  for (q = p; q != NULL; q = q->next) {
    /* Send the data from the pbuf to the interface, one pbuf at a time. The size of the data in each pbuf is kept in the ->len variable. */
    //实现发送data，从pbuf到网卡
    send data from(q->payload, q->len);
  }

  //实现触发发送
  signal that packet should be sent();

  MIB2_STATS_NETIF_ADD(netif, ifoutoctets, p->tot_len);
  if (((u8_t*)p->payload)[0] & 1) {
    /* broadcast or multicast packet*/
    MIB2_STATS_NETIF_INC(netif, ifoutnucastpkts);
  } else {
    /* unicast packet */
    MIB2_STATS_NETIF_INC(netif, ifoutucastpkts);
  }
  /* increase ifoutdiscards or ifouterrors on error */

#if ETH_PAD_SIZE
  pbuf_header(p, ETH_PAD_SIZE); /* reclaim the padding word */
#endif

  LINK_STATS_INC(link.xmit);

  return ERR_OK;
}

static struct pbuf *low_level_input(struct netif *netif)
{
  struct ethernetif *ethernetif = netif->state;
  struct pbuf *p, *q;
  u16_t len;

  //获得接收数据长度，赋值给len
  /* Obtain the size of the packet and put it into the "len" variable. */
  len = ;

#if ETH_PAD_SIZE
  len += ETH_PAD_SIZE; /* allow room for Ethernet padding */
#endif

  /* We allocate a pbuf chain of pbufs from the pool. */
  p = pbuf_alloc(PBUF_RAW, len, PBUF_POOL);

  if (p != NULL) {

#if ETH_PAD_SIZE
    pbuf_header(p, -ETH_PAD_SIZE); /* drop the padding word */
#endif

    /* We iterate over the pbuf chain until we have read the entire packet into the pbuf. */
    for (q = p; q != NULL; q = q->next) {
      /* Read enough bytes to fill this pbuf in the chain. The available data in the pbuf is given by the q->len variable. This does not necessarily have to be a memcpy, you can also preallocate pbufs for a DMA-enabled MAC and after receiving truncate it to the actually received size. In this case, ensure the tot_len member of the pbuf is the sum of the chained pbuf len members. */
      //实现读入数据
      read data into(q->payload, q->len);
    }
    //实现数据已经读取Ack
    acknowledge that packet has been read();

    MIB2_STATS_NETIF_ADD(netif, ifinoctets, p->tot_len);
    if (((u8_t*)p->payload)[0] & 1) {
      /* broadcast or multicast packet*/
      MIB2_STATS_NETIF_INC(netif, ifinnucastpkts);
    } else {
      /* unicast packet*/
      MIB2_STATS_NETIF_INC(netif, ifinucastpkts);
    }
#if ETH_PAD_SIZE
    pbuf_header(p, ETH_PAD_SIZE); /* reclaim the padding word */
#endif

    LINK_STATS_INC(link.recv);
  } else {
    //实现丢弃数据包
    drop packet();
    LINK_STATS_INC(link.memerr);
    LINK_STATS_INC(link.drop);
    MIB2_STATS_NETIF_INC(netif, ifindiscards);
  }

  return p;
}
```

### Porting Examples

lwIP还提供了一些[移植示例](http://lwip.wikia.com/wiki/Available_device_drivers)。基于Cortex-M3平台的[示例](http://scaprile.ldir.com.ar/cms/category/os/lwip-port/)，示例采用lwIP源码版本v1.4.1，可以参考其cc.h和sys_arch.h的修改；而另一个示例LwIP_TCP_Echo_Server/LwIP_HTTP_Server_Netconn_RTOS里面有关于driver的参考示例。
