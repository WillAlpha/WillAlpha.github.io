---
layout: post
title: "lwIP学习笔记-源码"
categories: TCP/IP
tags: Embedded TCP/IP lwIP
author: Will
---

* content
{:toc}


lwIP的源码可以到官网的Source Code页面[下载](http://git.savannah.nongnu.org/cgit/lwip.git)，下载v2.0.1 release版本，关于lwIP的源码设计文档，可以参考《Design and Implementation of the lwIP tcpIP stack.pdf》，另外官网的wiki的提供给开发者的文档[Architectural Rx flow](http://images2.wikia.nocookie.net/lwip/images/a/a9/Rx_flow.pdf)，也非常值得参考。


## 概述

lwIP的设计与常见的一些TCP/IP协议栈有些不同，如Linux的TCP/IP协议栈，在设计上严格区分应用层和底层协议栈之间的交互，底层的协议栈与操作系统融合，是操作系统的一部分，应用层需要调用操作系统提供的接口来实现与协议栈之间的交互，而且应用层也会抽象出一层供应用层调用的TCP/IP接口，而且在内存使用上应用层和底层协议栈也是分开的，因此存在数据的copy。lwIP一般用于实时系统，没有严格的应用层与操作系统层的区分，TCP/IP协议栈和应用层可以共享内存，因此不存在数据的copy，效率更高，消耗资源更少。
lwIP在实时系统中将TCP/IP协议栈的处理放入到一个单独的任务中，应用层的处理也可以放入这一个任务，也可以单独另外创建一个任务。lwIP中所有与操作系统相关的部分全部封装成了单独的函数或宏定义，开发者在移植的时候根据具体的操作系统来实现或重定义。因此，lwIP非常容易移植到各种版本的实时操作系统上。
lwIP是针对于嵌入式实时系统设计的TCP/IP协议栈，值得去研究一下它的设计理念和源码。

## Buffer and Memory Management

### pbufs

src/core/pbuf.c中的pbuf_alloc()和pbuf_free()

```c
typedef enum {
  //传输层header，如tcp/udp，如调用udp_send()
  PBUF_TRANSPORT,
  //IP层header，如调用raw_send()
  PBUF_IP,
  //数据链路层header，如调用ethernet_output()
  PBUF_LINK,
  //额外的ethernet header
  PBUF_RAW_TX,
  //网卡驱动层，如调用netif->input()
  PBUF_RAW
} pbuf_layer;

typedef enum {
  //pbuf的data存储在RAM，常用于Tx，pbuf数据结构和data分配在连续的RAM内存
  PBUF_RAM,
  //pbuf的data存储在ROM，pbuf数据结构与data分离在不同区域，pbuf->payload指向data，而且payload不会被copy
  PBUF_ROM,
  //类似于PBUF_ROM，data存储在RAM，但payload可能被复写
  PBUF_REF,
  //payload指向RAM，从内存pool中分配，一般用于Rx，类似于PBUF_RAM，pbuf数据结构和data分配在连续的RAM内存。不能用于Tx！
  PBUF_POOL
} pbuf_type;

struct pbuf *pbuf_alloc(pbuf_layer layer, u16_t length, pbuf_type type)
{
  ...

  //根据pbuf_layer的类型计算包头长度offset
  switch (layer) {
    ...
  }

  //根据pbuf_type类型具体进行内存分配
  switch (type) {
    case PBUF_POOL:
      //从内存pool中分配，调用memp_malloc()
      p = (struct pbuf *)memp_malloc(MEMP_PBUF_POOL);
      ...
      //payload直接指向data，内存对齐，跳过pbuf数据结构和offset
      p->payload = LWIP_MEM_ALIGN((void *)((u8_t *)p + (SIZEOF_STRUCT_PBUF + offset)));
      ...
      //一个包可能会被分割成多个包，tot_len是整个pbuf chain长度
      p->tot_len = length;
      //此pbuf包长
      p->len = LWIP_MIN(length, PBUF_POOL_BUFSIZE_ALIGNED - LWIP_MEM_ALIGN_SIZE(offset));
      ...
      p->ref = 1;
      r = p;
      //剩余长度
      rem_len = length - p->len;
      //循环链接成pbuf chain链表
      while (rem_len > 0) {
        //再分配一个pbuf
        q = (struct pbuf *)memp_malloc(MEMP_PBUF_POOL);
        ...
        //链表链接成pbuf chain
        r->next = q;
        q->tot_len = (u16_t)rem_len;
        q->len = LWIP_MIN((u16_t)rem_len, PBUF_POOL_BUFSIZE_ALIGNED);
        //payload指向data，已经没有header offset
        q->payload = (void *)((u8_t *)q + SIZEOF_STRUCT_PBUF);
        q->ref = 1;
        rem_len -= q->len;
        r = q;
      }
      break;

    case PBUF_RAM:
      //从堆内存中直接分配，pbuf数据结构和data在连续的RAM内存，没有形成pbuf chain
      p = (struct pbuf*)mem_malloc(LWIP_MEM_ALIGN_SIZE(SIZEOF_STRUCT_PBUF + offset) + LWIP_MEM_ALIGN_SIZE(length));
      ...
      p->payload = LWIP_MEM_ALIGN((void *)((u8_t *)p + SIZEOF_STRUCT_PBUF + offset));
      p->len = p->tot_len = length;
      p->next = NULL;
      p->type = type;
      ...
      break;

    case PBUF_ROM:
    case PBUF_REF:
      //从内存pool中分配，只分配pbuf数据结构大小的内存，payload后期再赋值，指向分离的data
      p = (struct pbuf *)memp_malloc(MEMP_PBUF);
      ...
      p->payload = NULL;
      p->len = p->tot_len = length;
      p->next = NULL;
      p->type = type;
      break;

    ...
  }

  p->ref = 1;
  p->flags = 0;

  return p;
}
```

通过pbuf_alloc()可见pbuf分四种类型：PBUF_POOL，PBUF_RAM，PBUF_REF，PBUF_ROM。

* PBUF_POOL

![lwIP_pbuf_pool_model]({{ "/images/lwip_pbuf_pool_model.png" | prepend:site.baseurl }})

PBUF_POOL从内存pool中分配，固定长度分配，分配的每一个pbuf包都是连续内存，如果包长超过一个pbuf固定分配长度，则拆分成多个pbuf包，然后形成pbuf chain，一般用于Rx，即device drivers接收网络数据包

* PBUF_RAM

![lwIP_pbuf_ram_model]({{ "/images/lwip_pbuf_ram_model.png" | prepend:site.baseurl }})

PBUF_RAM从内存堆中分配，4种类型中唯一分配方式不同的一个，非固定长度，分配在连续内存上，一般用于Tx

* PBUF_REF & PBUF_ROM

![lwIP_pbuf_ref_rom_model]({{ "/images/lwip_pbuf_ref_rom_model.png" | prepend:site.baseurl }})

PBUF_REF和PBUF_ROM分配相同，在内存pool中分配，只分配pbuf数据结构大小，payload为NULL，将来指向data内存，pbuf数据结构和data分离，常用于应用

接着再看一下pbuf_free()

```c
/* 官方示例：根据引用次数free()前后的变化
 * 1->2->3 becomes ...1->3
 * 3->3->3 becomes 2->3->3
 * 1->1->2 becomes ......1
 * 2->1->1 becomes 1->1->1
 * 1->1->1 becomes .......
 */
u8_t pbuf_free(struct pbuf *p)
{
  ...

  count = 0;
  while (p != NULL) {
    ...
    ref = --(p->ref);
    if (ref == 0) { //此pbuf不再被其他pbuf引用
      q = p->next;
      type = p->type;
      ...
      if (type == PBUF_POOL) {
        memp_free(MEMP_PBUF_POOL, p);
      } else if (type == PBUF_ROM || type == PBUF_REF) {
        memp_free(MEMP_PBUF, p);
      } else {
        mem_free(p);
      }
      ...
      count++;
      p = q;
    } else {
      p = NULL; //此pbuf还在被其他pbuf引用，不能free，退出
    }
  }

  return count;
}
```

从pbuf_free()可以看出pbuf->ref用于记录此pbuf被引用的次数，如果不再被引用了则free掉，如果还有被引用，则减少引用记录次数，不free然后退出；如果是pbuf chain，则在遇到第一个不能free掉的pbuf后直接退出，不再继续遍历。

### memeory management

通过上面的pbuf源码，可以看出lwIP有2种内存分配方式，即memp_malloc()/memp_free()和mem_malloc()/mem_free()。mem_malloc()/mem_free()与我们常用的堆分配释放类似，lwIP里面主要应对不定长的Tx时的pbuf，可以使用标准c库里的堆管理，也可以使用静态内存池，也可以使用自定义的堆管理，与前面FreeRTOS的heap_4类似的堆管理，简洁高效，尽量避免内存碎片。主要看一下memp_malloc()/memp_free()，用来应对定长的pbuf，定长的内存管理更加的简洁，效率更高，不会产生内存碎片，但是可能存在浪费空间的问题(PBUF_POOL)，总要有所取舍，特别在接收时使用这种内存分配策略。

```c
#define LWIP_MEMPOOL_DECLARE(name,num,size,desc) \
  LWIP_DECLARE_MEMORY_ALIGNED(memp_memory_ ## name ## _base, ((num) * (MEMP_SIZE + MEMP_ALIGN_SIZE(size)))); \
  \ LWIP_MEMPOOL_DECLARE_STATS_INSTANCE(memp_stats_ ## name) \
  \ static struct memp *memp_tab_ ## name; \
  \ const struct memp_desc memp_ ## name = { \
    DECLARE_LWIP_MEMPOOL_DESC(desc) \
    LWIP_MEMPOOL_DECLARE_STATS_REFERENCE(memp_stats_ ## name) \
    LWIP_MEM_ALIGN_SIZE(size), \
    (num), \
    memp_memory_ ## name ## _base, \
    &memp_tab_ ## name \
  };

#define LWIP_MEMPOOL(name,num,size,desc) LWIP_MEMPOOL_DECLARE(name,num,size,desc)
#include "lwip/priv/memp_std.h"

const struct memp_desc* const memp_pools[MEMP_MAX] = {
#define LWIP_MEMPOOL(name,num,size,desc) &memp_ ## name,
#include "lwip/priv/memp_std.h"
};
```

上面的源码是memp相关的定义，摘录自memp.c/memp.h，另外再结合memp_std.h文件就可以明白作者的代码原理，编译阶段预处理将宏定义展开后，还原出c语言源码，而增删改查则通过头文件memp_std.h就可以完成，非常简洁的源码设计，值得学习！

```c
...

#if LWIP_UDP //通过宏LWIP_UDP控制是否定义TCP相关的静态内存pool
LWIP_MEMPOOL(UDP_PCB, MEMP_NUM_UDP_PCB, sizeof(struct udp_pcb), "UDP_PCB")
#endif

#if LWIP_TCP //通过宏LWIP_TCP控制是否定义TCP相关的静态内存pool
LWIP_MEMPOOL(TCP_PCB, MEMP_NUM_TCP_PCB, sizeof(struct tcp_pcb), "TCP_PCB")
LWIP_MEMPOOL(TCP_PCB_LISTEN, MEMP_NUM_TCP_PCB_LISTEN, sizeof(struct tcp_pcb_listen), "TCP_PCB_LISTEN")
LWIP_MEMPOOL(TCP_SEG, MEMP_NUM_TCP_SEG, sizeof(struct tcp_seg), "TCP_SEG")
#endif

...

//PBUF_REF/ROM/POOL相关的静态内存pool定义
LWIP_PBUF_MEMPOOL(PBUF, MEMP_NUM_PBUF, 0, "PBUF_REF/ROM")
LWIP_PBUF_MEMPOOL(PBUF_POOL, PBUF_POOL_SIZE, PBUF_POOL_BUFSIZE, "PBUF_POOL")

...

#if MEMP_USE_CUSTOM_POOLS //如果启用自定义静态内存pool来替换mem.c里面的内存管理，还需定义头文件lwippools.h
#include "lwippools.h"
#endif

//每次#include memp_std.h，根据源码的#define，将本次的#uddef
#undef LWIP_MEMPOOL
#undef LWIP_MALLOC_MEMPOOL
#undef LWIP_MALLOC_MEMPOOL_START
#undef LWIP_MALLOC_MEMPOOL_END
#undef LWIP_PBUF_MEMPOOL
```

通过memp_std.h里的宏定义，在编译时确定静态内存pool大小，整片静态内存，分割成定长的内存块，再看pbuf.c中的memp_malloc(MEMP_PBUF_POOL)和memp_malloc(MEMP_PBUF)，其中MEMP_PBUF_POOL/MEMP_PBUF是enum memp_t定义的枚举类型(源码如下所示)，编译阶段展开，通过这个枚举类型在相应的连续整片的静态内存pool中分配固定长度的内存块，这些内存块形成链表的结构，定长的内存块链表在分配释放上是非常快的，而且不产生内存碎片，只是有时候有可能有些浪费内存，但是在此更倾向于效率。

```c
#define LWIP_MEMPOOL(name,num,size,desc)
#include "lwip/priv/memp_std.h"

typedef enum {
#define LWIP_MEMPOOL(name,num,size,desc)  MEMP_##name,
#include "lwip/priv/memp_std.h"
  MEMP_MAX
} memp_t;
```

再看几个宏定义：

```c
#define MEMP_MEM_MALLOC 0 //采用memp_malloc/memp_free，如果此宏定义定义为1，则采用mem_malloc/mem_free

#define MEM_USE_POOLS 0 //采用mem_malloc/mem_free自己的内存管理方式，如果定义为1，则采用静态内存pool这种类型的方式，此时还需要定义另一个宏和头文件

#define MEMP_USE_CUSTOM_POOLS 0 //如上所示，如果mem_malloc/mem_free采用静态内存pool的方式则需要将此宏定义为1，而且通过memp_std.h头文件可知，还需定义头文件lwippools.h，定义静态内存pool
```

## netif

lwIP使用数据结构netif来描述一个网络接口

```c
struct netif {
  struct netif *next; //指向下一个netif

#if LWIP_IPV4
  ip_addr_t ip_addr; //ipv4的IP地址，子网掩码，网关设置
  ip_addr_t netmask;
  ip_addr_t gw;
#endif
  ...
  netif_input_fn input; //接收函数，网卡驱动调用，送给tcp/ip stack
#if LWIP_IPV4
  netif_output_fn output; //发送函数，IP层要发送数据调用
#endif
  netif_linkoutput_fn linkoutput; //ethernet_output()发送调用，arp调用
  ...
  void *state; //指向网卡状态，可以由网卡驱动修改，可由开发者自定义的ethernetif
#ifdef netif_get_client_data
  void* client_data[LWIP_NETIF_CLIENT_DATA_INDEX_MAX + LWIP_NUM_NETIF_CLIENT_DATA];
#endif
  ...
  u16_t mtu; //maximum transfer unit (in bytes)
  u8_t hwaddr_len; //mac地址长度
  u8_t hwaddr[NETIF_MAX_HWADDR_LEN]; //mac地址
  u8_t flags;
  char name[2]; //网络接口类型描述
  u8_t num; //网络接口个数
  ...
#if LWIP_IPV4 && LWIP_IGMP
  netif_igmp_mac_filter_fn igmp_mac_filter;
#endif
  ...
#if ENABLE_LOOPBACK
  struct pbuf *loop_first;
  struct pbuf *loop_last;
#if LWIP_LOOPBACK_MAX_PBUFS
  u16_t loop_cnt_current;
#endif
#endif
};
```

网卡驱动初始化时用到此数据结构，参看一下测试程序的初始化就更直观了test\fuzz\fuzz.c，但是还不是具体的设备，只是一个测试程序，参考一个具体的例子

```c
  ...
  struct netif net_test; //网络接口定义
  ip4_addr_t addr;
  ip4_addr_t netmask;
  ip4_addr_t gw;
  u8_t pktbuf[2000];
  size_t len;

  lwip_init(); //lwip初始化

  IP4_ADDR(&addr, 172, 30, 115, 84);
  IP4_ADDR(&netmask, 255, 255, 255, 0);
  IP4_ADDR(&gw, 172, 30, 115, 1);

  netif_add(&net_test, &addr, &netmask, &gw, &net_test, testif_init, ethernet_input); //关注初始化函数，input函数
  netif_set_up(&net_test);
  ...
```

网卡的初始化的驱动文件ethernetif.c是lwIP提供的一个网卡驱动模板，err_t ethernetif_init(struct netif *netif)，更底层的初始化函数static void low_level_init(struct netif *netif)，以及相应的底层input/output
