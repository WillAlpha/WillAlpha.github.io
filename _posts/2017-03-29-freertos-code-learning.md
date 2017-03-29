---
layout: post
title: "RTOS学习笔记-FreeRTOS源码"
categories: RTOS
tags: ARM RTOS FreeRTOS Cortex-M
author: Will
---

* content
{:toc}


通过FreeRTOS官网资料，已经可以很好的使用FreeRTOS了，再深入的理解就需要深入到源码层，接下来就阅读源码。

FreeRTOS需要关注的源码：

```
//核心代码
list.c
task.c
queue.c

//平台相关
port.c
//已经分析过，Memory Management相关
heap_x.c

//非核心代码
event_groups.c
timers.c
croutine.c
```

