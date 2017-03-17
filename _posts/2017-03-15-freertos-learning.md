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

## Advanced

### FreeRTOS宏定义

FreeRTOS宏定义主要分布在3个头文件中：

* FreeRTOSConfig.h  //user application相关的设置
* FreeRTOS.h  //内核相关的设置，如果user没有定义的宏在这里给出默认值
* portmacro.h  //移植平台相关的宏定义

configUSE_PREEMPTION：设置为1，抢占式调度；0为协作式调度，即时间片轮转调度。一般都设置成1

configUSE_PORT_OPTIMISED_TASK_SELECTION：见上面Tasks里面有分析

configUSE_TICKLESS_IDLE：设置为1，低功耗tickless模式，即系统1ms节拍定时中断在低功耗时停止，唤醒校正后继续，这样做的主要考虑是节拍中断频率过高也带来过多的功耗；设置为0则节拍中断持续不关断。关于低功耗后面还有专门的学习

configUSE_IDLE_HOOK：见上面Idle Task，设置为1启用idle task的hook函数，idle task的工作，设置为0不启用

configUSE_MALLOC_FAILED_HOOK：分配内存失败，malloc返回NULL时，调用的hook函数，前提需要用户在移植FreeRTOS后，使用堆方案为FreeRTOS提供的heap_1~5.h中的任意一个方案。设置为1需调用，用户需要定义；设置为0不需要调用。函数原型：

```c
void vApplicationMallocFailedHook(void);
```

configUSE_DAEMON_TASK_STARTUP_HOOK：默认设置为0

configUSE_TICK_HOOK:设置为1，在节拍中断的处理函数里面会执行这个hook函数，设置为0则不执行。一般没什么需要设置为0。函数原型：

```c
void vApplicationTickHook(void); 
```

configCPU_CLOCK_HZ:写入正确的系统时钟频率，后面相应的时钟节拍也要基于此计算

configTICK_RATE_HZ：时钟节拍频率，即1秒钟多少拍，一般设置为1000

configMAX_PRIORITIES：设置有效任务优先级的最大数，详见上面Tasks里有详细介绍

configMINIMAL_STACK_SIZE：设置Idle Task的堆栈大小，一般以word为单位

configMAX_TASK_NAME_LEN：创建Task时，描述Task名字字符串长度

configUSE_TRACE_FACILITY：设置为1启动可视化跟踪调试，附加一些额外的数据结构和函数。STM32CubeMx工程里默认直接设置1

configUSE_STATS_FORMATTING_FUNCTIONS：配合前面的configUSE_TRACE_FACILITY / configGENERATE_RUN_TIME_STATS设置为1，而其也设置为1，则会打开几个函数(详见源码，默认设置为0)：

```c
static char *prvWriteNameToBuffer(char *pcBuffer, const char *pcTaskName)
void vTaskList(char * pcWriteBuffer)
void vTaskGetRunTimeStats(char *pcWriteBuffer)
```

configUSE_16_BIT_TICKS：时钟节拍计数器变量类型，设置为1则portTickType为16bit，而设置为0则为32bit，STM32设置成32bit，时钟计数器的最大时钟计数更大。

configIDLE_SHOULD_YIELD：详见Idle Task里的说明

configUSE_TASK_NOTIFICATIONS：设置为1则包含使能Task Notifications，设置为0则不包含，是能后每个Task额外多使用8个Bytes的RAM。新特性，是FreeRTOS解除task阻塞状态最快的方式。设置为1

configUSE_MUTEXES：设置为1包含使用Mutexes，设置为0则不包含使用

configUSE_RECURSIVE_MUTEXES：同上对Recursive Mutexes设置

configUSE_COUNTING_SEMAPHORES：同上对Counting Semaphores的设置

configCHECK_FOR_STACK_OVERFLOW：详见后面关于栈溢出的学习

configQUEUE_REGISTRY_SIZE：主要用于FreeRTOS内核调试，队列记录的2个目的:(1)GUI调试的时候用队列文本名简单的识别队列(2)包含调试器需要的Queues或Semaphores的记录信息。除了FreeRTOS内核调试之外没有任何其他的目的，此值定义最大记录长度，STM32F4xx里面默认设置成8

configUSE_QUEUE_SETS：设置成1使能queue set功能，0则不使能。默认设置为0

configUSE_TIME_SLICING：详见Task里的说明，默认设置成1

configUSE_NEWLIB_REENTRANT：设置为1则[newlib](http://sourceware.org/newlib/) reent数据结构在创建task时allocated，FreeRTOS默认不使用，需要user自己决定是否使用，并能完全掌握newlib的情况下自行使用。

configENABLE_BACKWARD_COMPATIBILITY：为了兼容v8.0.0之前的版本定义的一些宏，没有历史包袱可以不打开这个宏

configNUM_THREAD_LOCAL_STORAGE_POINTERS：详见后面关于Thread Local Storage Pointers的学习，默认为0

configSUPPORT_STATIC_ALLOCATION：默认为0，RTOS的对象只能在RTOS的heap上分配；设置成1，则需要用户提供额外的回调函数提供memory，vApplicationGetIdleTaskMemory和vApplicationGetTimerTaskMemory。详见后面的学习内容

configSUPPORT_DYNAMIC_ALLOCATION：默认设置为1，RTOS对象在RTOS heap的RAM里分配；设置为0需要通过回调函数提供额外memory。同上见后面的学习内容

configTOTAL_HEAP_SIZE：上面的宏设置为1，这个宏设置堆大小，STM32F4xx默认设置为15K

configAPPLICATION_ALLOCATED_HEAP：默认为0，如果设置为1，需要user自定义heap数组：

```c
uint8_t ucHeap[ configTOTAL_HEAP_SIZE ];
```

configGENERATE_RUN_TIME_STATS:默认为0，详见后面Run Time Stats

configUSE_CO_ROUTINES：默认为0，与co-routines相关

configMAX_CO_ROUTINE_PRIORITIES：STM32F4xx默认为2，32位MCU一般不用Co-routines

configUSE_TIMERS：默认为0，不打开Software Timers功能；1则不打开

configTIMER_TASK_PRIORITY：如果上面的宏打开，则需要设置

configTIMER_QUEUE_LENGTH：同上

configTIMER_TASK_STACK_DEPTH：同上

configKERNEL_INTERRUPT_PRIORITY &
configMAX_SYSCALL_INTERRUPT_PRIORITY &
configMAX_API_CALL_INTERRUPT_PRIORITY：
这几个宏比较关键，移植工作中需要小心关注，与平台相关性大。总终配置还是以ARM Cortex-M4内核为例，详见[RTOS-Cortex-M3-M4介绍](http://www.freertos.org/RTOS-Cortex-M3-M4.html)。

Coretex-M4需要配置configKERNEL_INTERRUPT_PRIORITY和configMAX_SYSCALL_INTERRUPT_PRIORITY，而configMAX_API_CALL_INTERRUPT_PRIORITY与configMAX_SYSCALL_INTERRUPT_PRIORITY等价，主要是应用于不同的平台移植时的兼容性上。

Cortex-M系列的中断机制与FreeRTOS配合非常紧密，但是各家的Cortex-M处理器在中断优先级寄存器的设计上不尽相同，Cortex-M最多允许256级可编程优先级，寄存器8bit，配置范围(0~0xff)，不过STM32F4xx系列使用了4bit，所以范围是(0~0xf)，共16级可编程优先级。这里还要注意一点，ARM的中断优先级是数字越低，优先级越高，最高优先级是0，而与一般的逻辑优先级的概念刚好与此相反。STM32F4xx系列优先级寄存器是4bit，占用的是8bit寄存器的高4bit，如下图所示，优先级5在寄存器中的配置，其他不使用的位全部置为1：

![Cortex-M_4bit_中断优先级配置寄存器示例]({{ "/images/cortex-m-priority-register-4-bits.png" | prepend:site.baseurl }})

这样就能更好的理解FreeRTOS里面的配置(STM32F4xx Cortex-M4)：

```c
/* Cortex-M specific definitions. STM32F4xx use high 4bit to register interrupt priority*/
#define configPRIO_BITS 4

/* The lowest interrupt priority that can be used in a call to a "set priority" function. */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY 15

/* The highest interrupt priority that can be used by any interrupt service routine that makes calls to interrupt safe FreeRTOS API functions.  DO NOT CALL INTERRUPT SAFE FREERTOS API FUNCTIONS FROM ANY INTERRUPT THAT HAS A HIGHER
PRIORITY THAN THIS! (higher priorities are lower numeric values. */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5

/* Interrupt priorities used by the kernel port layer itself.  These are generic to all Cortex-M ports, and do not rely on any particular library functions. */
#define configKERNEL_INTERRUPT_PRIORITY (configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS))
/* !!!! configMAX_SYSCALL_INTERRUPT_PRIORITY must not be set to zero !!!! See http://www.FreeRTOS.org/RTOS-Cortex-M3-M4.html. */
#define configMAX_SYSCALL_INTERRUPT_PRIORITY (configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS))
```

configKERNEL_INTERRUPT_PRIORITY一般配置成平台的最低优先级，FreeRTOS 内核就运行在这个优先级，configMAX_SYSCALL_INTERRUPT_PRIORITY配置为高优先级上限，这里配置成了5，那么更高的优先级(0~4)就被保留变成FreeRTOS不可屏蔽的中断优先级，这些更高优先级的中断里就不可以再调用FreeRTOS API，也完全不会受到FreeRTOS内核的影响，具有非常高的实时性，而中断优先级(5~15)就可以完全被FreeRTOS托管，应用程序应该使用这个中断优先级区间的优先级。

configASSERT：断言的定义，开发阶段打开，release后可以关闭，不关闭会增大代码空间，带来性能损失。

configINCLUDE_APPLICATION_DEFINED_PRIVILEGED_FUNCTIONS：默认为0，此宏定义仅与MPU功能相关，在开发阶段使用，release阶段需关闭。


INCLUDE_xxx：以此前缀开头的宏用于决定是否将此API编译进项目工程，设置为1则进行编译，设置为0表示不编译。

### Low Power Support