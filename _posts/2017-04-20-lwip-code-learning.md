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

通过上面的pbuf源码，可以看出lwIP有2种内存分配方式，即memp_malloc()/memp_free()和mem_malloc()/mem_free()。mem_malloc()/mem_free()与我们常用的堆分配释放类似，lwIP里面主要应对不定长的Tx时的pbuf，可以使用标准c库里的堆管理，也可以使用静态内存池，也可以使用自定义的堆管理，与前面FreeRTOS的heap_4类似的堆管理，简洁高效，尽量避免内存碎片。主要看一下memp_malloc()/memp_free()，用来应对定长的pbuf，定长的内存管理更加的简洁，效率更高，不会产生内存碎片，但是可能存在浪费空间的问题(PBUF_POOL)，总要有所取舍。



