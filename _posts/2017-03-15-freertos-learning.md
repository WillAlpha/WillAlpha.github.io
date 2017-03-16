---
layout: post
title: "RTOS学习笔记-FreeRTOS"
categories: RTOS
tags: ARM RTOS FreeRTOS Cortex-M
author: Will
---

* content
{:toc}


FreeRTOS作为开源实时嵌入式操作系统，近几年RTOS使用量上一直名列前茅，[《ARM学习笔记-STM32》](https://willalpha.github.io/2017/02/19/arm-stm32-learning/)中已经初步接触到了FreeRTOS的移植以及简单的使用，对FreeRTOS的深入学习可以通过[FreeRTOS官网](http://www.freertos.org/about-RTOS.html)提供的资料继续学习，左侧导航栏从简介到Basic知识点再到Advanced进阶学习，FreeRTOS源码版本v9.0.0，平台选择STM32F4xx（ARM Cortex-M4）。

## Basic

### Tasks & Co-routines

FreeRTOS提供了Co-routines，中文名大家都称为协程，可以单独使用也可以与Task混合使用，Co-routines的stack是共享模式，不像每个tasks那样都有自己的单独的stack，主要是针对RAM非常稀缺的使用场景，目前32位的MCU应该不会再使用，因此先不对其做过多的深入，主要看tasks。Tasks其实都比较熟悉了，借用官网的状态图一展：

![FreeRTOS_tasks状态图]({{ "/images/freertos_tskstate.gif" | prepend:site.baseurl }})

Tasks的优先级设置为0~(configMAX_PRIORITIES - 1)，configMAX_PRIORITIES是在FreeRTOSConfig.h中配置的，这个宏定义不能随意定义，如果configUSE_PORT_OPTIMISED_TASK_SELECTION宏定义为1，则configMAX_PRIORITIES宏定义必须小于32，除此之外可以设置成任意值，但是configMAX_PRIORITIES的值越大，对RAM消耗越大，根据项目的需求设置成一个合理的值最佳。

configUSE_PORT_OPTIMISED_TASK_SELECTION宏定义针对某些处理器的特殊指令，如ARM Cortex-M的`CLZ`(Count Leading Zeros)前导零计数指令，这条指令可以得到被操作的32bit数从高位起0的个数，如被操作数的bit[31] = 1，则返回的结果是0，如果操作数为0，返回结果为32，详见指令：

```
CLZ{<cond>} <Rd>，<Rm>

指令伪代码：
If Rm==0
  Rd=32
Else
  Rd=31 - (bit position of most significant “1” in Rm)
```

FreeRTOS里计算当前处于任务Ready队列里的最高优先级的任务：

```c
/* Find the highest priority queue that contains ready tasks. */
#define portGET_HIGHEST_PRIORITY(uxTopPriority, uxReadyPriorities)  uxTopPriority=(31-__clz((uxReadyPriorities)))
```

由此也可以理解为什么使用这个指令优先级必须小于32。当然，这条指令还可以用于math计算等。

当前运行的task一般为满足运行条件的最高优先级任务。如果宏configUSE_TIME_SLICING未定义或定义为1，则相同优先级的任务之间采取时间片轮转调度。

Idle Task是FreeRTOS默认的最低优先级task，如果有其他任务也设置成了最低优先级，这个时候宏定义configIDLE_SHOULD_YIELD设置为1的作用是，如果其他的与idle task相同优先级的task处于了Ready状态，则idle task以最快的速度让另一个任务得到CPU并执行。但是，实际项目中Idle Task的作用更多的是考虑低功耗，因此一般都会将最低优先级保留给Idle Task。

Idle Task创建可以将宏定义configUSE_IDLE_HOOK设置为1，定义并实现Idle Task的钩子函数：

```c
void vApplicationIdleHook( void );
```

或者单独定义一个最低优先级的任务也是可行的，FreeRTOS更推荐前一种做法。

### Queues,Semaphores,Mutexes

Queues用于tasks间的通讯，简单而灵活，发送到队列的消息是Copy进队列的，队列里做了缓存，这样tasks间的传递更加灵活，也可以传递指针，自定义消息数据结构和memory poll，对携带的消息格式和大小没有限制，一个队列可以接收各式消息，适用于MPU功能的场景，在发送消息时会提升MCU的权限，提升权限后就可以访问任意的存储区域，在中断处理函数中使用带`FromISR`结尾的API。

Semaphores与Mutexes：
* Semaphores分为Binary Semaphores和Counting Semaphores
* Mutexes分为Mutexes和Recursive Mutexes
* Semaphores用于任务间同步，tasks之间或者tasks与isr之间；Mutexes用于资源互斥，是保护某资源的token
* Semaphores在某个task或isr中give，在另一个中take；Mutexes要在某个task内take & give，Binary Semaphores也可类似使用。
* Mutexes具有优先级继承，当低优先级任务获得Mutexes运行时，自动将优先级升至与等待此Mutexes的最高优先级的任务一致，不能解决优先级反转问题，但是可以最大化的减少等待时间；Semaphores不具备此特性
* Semaphores与Mutexes的Create API不同，但是take/give的API相同，Mutex不能用于中断服务程序，没有带`FromISR`结尾的API

### Direct To Task Notifications

Task Notifications相较于Semaphores/Mutexes/Event Groups在解除阻塞上有明显的性能优势，官方给出的结论：
* 45% faster and uses less RAM 相较于Semaphores
* significant performance benefits 相较于Event Groups

通过宏定义configUSE_TASK_NOTIFICATIONS置为1打开Task Notifications功能。使用限制：
* 只能1个task接收Task Notifications
* 等待Task Notifications的task可以进入Blocked状态，但是发送Notifications的task不会因为无法立即发出Notifications而进入Blocked状态

### Event Groups

宏定义configUSE_16_BIT_TICKS设置为1，则Event Groups具有8bit，如果设置为0，则有24bit。可以同时设置1或多个bit位，清除1或多个bit位，task进入Blocked状态等待某1个或多个bit位置位；event groups可以用于tasks的同步。

使用Event Groups的需要克服的条件：
* 避免产生race conditions。设置、测试和清除event bit是atomic原子操作
* 避免不确定性，event groups不知道有多少tasks因为event groups进入阻塞状态，也不知道event bit改变后有多少tasks会进入运行状态。因此在task里设置一个event bit时，FreeRTOS启动调度锁机制，确保中断使能状态；在isr内试图设置event bit时，启动中断延迟机制，推迟设置event bit的动作。

### Example Code

Example在FreeRTOS源码里的路径是：FreeRTOS/Demo/Common/Minimal
