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

这个钩子函数是Idle Task流程中留给开发者实现的函数，开发者可以自定义在Idle Task里的执行内容，提醒开发者这个函数里面不可以有任何Block的操作，通过FreeRTOS的源码，task.c里面的static portTASK_FUNCTION(prvIdleTask, pvParameters)定义的Idle Task函数prvIdleTask的内容可以看到vApplicationIdleHook()的执行在tickless省电模式的前面，低功耗的流程在宏configUSE_TICKLESS_IDLE设置为非0的情况下执行，具体后面低功耗小节会详细介绍。

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

FreeRTOS默认已经实现了Idle Task，并在Idle Task里也给出了低功耗的流程，需要开发者添加一些自己应用场景相关的一些操作即可使用，其他就是一些宏开关的设置。官方给出的低功耗解读请参考[low-power-tickless-rtos](http://www.freertos.org/low-power-tickless-rtos.html)，而与ARM Cortex-M系列相关的请参考[low-power-ARM-CortexM-rtos](http://www.freertos.org/low-power-ARM-cortex-rtos.html)

FreeRTOS用宏configUSE_TICKLESS_IDLE控制系统是否进入Tickless Idle Mode，FreeRTOS使用一个定时器产生操作系统Tick，一般设置为1ms产生1个Tick中断，如果系统进入低功耗以后，依然产生这个1ms中断，假如很长一段时间都没有高优先级任务需要执行，那么这1ms中断频繁从低功耗模式醒来也会带来很可观的功耗，因此有了这个Tickless模式。Tickless模式的原理就是进入低功耗前关闭1ms的中断，而是设置一个本次低功耗的阈值，无论是这个阈值时间到了从低功耗被唤醒，还是被其他的中断或事件唤醒，再重新启动1ms中断和修正操作系统的TickCount值，以此来达到低功耗情况下尽可能少的被唤醒，也是一种更好的省电策略。

configUSE_TICKLESS_IDLE被置为1，FreeRTOS就可以在系统空闲并进入Idle Task之后进入到Tickless的处理流程，Idle Task里的源码在task.c里面：

```c
#if ( configUSE_TICKLESS_IDLE != 0 )
...
 //开发者自定义，增加此时的trace调试相关
 traceLOW_POWER_IDLE_BEGIN();
 //port.c中定义了Tickless低功耗的具体实现
 portSUPPRESS_TICKS_AND_SLEEP( xExpectedIdleTime );
 //开发者自定义，增加此时的trace调试相关
 traceLOW_POWER_IDLE_END();
...
#endif /* configUSE_TICKLESS_IDLE */
```

关键Tickless低功耗模式参见port.c里的源码：

```c
__weak void vPortSuppressTicksAndSleep( TickType_t xExpectedIdleTime )
{
 uint32_t ulReloadValue, ulCompleteTickPeriods, ulCompletedSysTickDecrements, ulSysTickCTRL;
 TickType_t xModifiableIdleTime;

 //ARM Systick提供Tick，Systick寄存器24bit，与Core时钟同频
 //FreeRTOSConfig.h configSYSTICK_CLOCK_HZ=configCPU_CLOCK_HZ
 //确保Systick的赋值不会溢出
 if( xExpectedIdleTime > xMaximumPossibleSuppressedTicks )
 {
  xExpectedIdleTime = xMaximumPossibleSuppressedTicks;
 }

 //停止Systick
 portNVIC_SYSTICK_CTRL_REG &= ~portNVIC_SYSTICK_ENABLE_BIT;
 //计算Systick的Reload Tick数
 ulReloadValue = portNVIC_SYSTICK_CURRENT_VALUE_REG + ( ulTimerCountsForOneTick * ( xExpectedIdleTime - 1UL )
 if( ulReloadValue > ulStoppedTimerCompensation )
 {
   ulReloadValue -= ulStoppedTimerCompensation;
 }

 //进入临界区
 //不使用taskENTER_CRITICAL()因其会退出省电模式
 __disable_irq();
 __dsb( portSY_FULL_READ_WRITE );
 __isb( portSY_FULL_READ_WRITE );

 //如果不符合进入低功耗的条件，暂时先不进低功耗
 if( eTaskConfirmSleepModeStatus() == eAbortSleep )
 {
   portNVIC_SYSTICK_LOAD_REG = portNVIC_SYSTICK_CURRENT_VALUE_REG;
   portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;
   portNVIC_SYSTICK_LOAD_REG = ulTimerCountsForOneTick - 1UL;
   __enable_irq();
 }
 else //进入低功耗的处理流程
 {
   //Systick设置成重载的值，即预期进入低功耗的计算出的合适的阈值
   portNVIC_SYSTICK_LOAD_REG = ulReloadValue;
   portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;
   portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;
   xModifiableIdleTime = xExpectedIdleTime;

   //开发者可以在下面这个宏定义自己的低功耗处理方式
   //而不用执行下面默认的进入低功耗的流程
   //也可以在这个宏定义里添加其他时钟/电源/外设的处理
   configPRE_SLEEP_PROCESSING( xModifiableIdleTime );
   if( xModifiableIdleTime > 0 )
   {
     __dsb( portSY_FULL_READ_WRITE );
     __wfi(); //WFI指令进入ARM的Sleep模式，任意中断可以唤醒
     __isb( portSY_FULL_READ_WRITE );
   }
   configPOST_SLEEP_PROCESSING( xExpectedIdleTime );

   ulSysTickCTRL = portNVIC_SYSTICK_CTRL_REG;
   portNVIC_SYSTICK_CTRL_REG = ( ulSysTickCTRL & ~portNVIC_SYSTICK_ENABLE_BIT );
   __enable_irq();

   //计算FreeRTOS的Tick计数的补偿值
   if( ( ulSysTickCTRL & portNVIC_SYSTICK_COUNT_FLAG_BIT ) != 0 )
   {
     //Systick计时到了产生的中断唤醒情况下的补偿计算
     ...
   }
   else
   {
     //其他中断唤醒情况下的补偿计算，此时Systick计时未到
     ...
   }

   //更新FreeRTOS的Tick计数xTickCount，启动Systick的1ms的Tick中断
   portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;
   portENTER_CRITICAL();
   {
     portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;
     vTaskStepTick( ulCompleteTickPeriods );
     portNVIC_SYSTICK_LOAD_REG = ulTimerCountsForOneTick - 1UL;
   }
   portEXIT_CRITICAL();
 }
}
```

以上就是FreeRTOS默认的低功耗处理流程，系统使用ARM Cortex-M提供的Systick定时器产生Tick时钟，1ms中断，进入低功耗后停止Systick的1ms中断，重新加载一个更长的时间周期，这个期间FreeRTOS的Tick计数也不更新了，系统唤醒后再重新开启Systick的1ms中断，并补偿休眠期间RTOS的Tick计数。很多开发者为了达到更加省电的目的，期望有更长的睡眠时间，即上面Systick的加载值更大，但是Systick是24Bit的(最大值0xffffff)，而且其使用的时钟频率与Core相同(160MHz)，因此由这2个参数就决定了低功耗使用Systick计时的每次休眠时间有一个上限值，大约100ms，睡眠周期太短，开发者就会考虑使用其他定时器替代Systick，如32Bit的定时器(最大值0xffffffff)，这样上限值就扩大到了25s左右。再激进一点，如果在进入低功耗的时候，采用一个32Bit定时器而且是低频率的在睡眠期间做计时，醒来后再重新开启Systick和补偿RTOS计数，是不是就能够获得更长的睡眠周期^_^

FreeRTOS针对ARM Cortex-M也给出了相应[说明](http://www.freertos.org/low-power-ARM-cortex-rtos.html)，Systick定时器是FreeRTOS默认的为ARM Cortex-M处理器提供Tick中断的24Bit定时器，其Systick工作频率：

* 可以与Core同频，FreeRTOSConfig.h里设置宏configSYSTICK_CLOCK_HZ与configCPU_CLOCK_HZ相同【默认】
* 为了获得更长的休眠周期，也可以与Core不同频，单独设置宏configSYSTICK_CLOCK_HZ为Systick的时钟频率

FreeRTOS从v7.3.0版本之后，支持开发者可以不使用Systick定时器而改用其他定时器为RTOS提供Tick中断：

* 重新定义函数void vPortSetupTimerInterrupt(void)，替换成其他定时器产生RTOS的Tick中断，这个函数在vTaskStartScheduler()调用时被调用，初始化定时器并使能Tick中断，Tick周期参照宏configTICK_RATE_HZ定义。另外注意configOVERRIDE_DEFAULT_TICK_CONFIGURATION设置为1，原来的vPortSetupTimerInterrupt()就被注释掉，需要同名再创建一个新的函数
* 相应的新定时器的中断处理函数也要定义，默认Systick的中断处理函数是xPortSysTickHandler()
* 注意CMSIS标准命名SysTick_Handler()，在启动文件*.s汇编文件里用的这个命名为RTOS的Tick中断处理函数，使用新的定时器为RTOS提供Tick，不要与Systick定时器的中断处理程序搞混了，特别是命名上需要注意

在执行WFI指令前后的两个宏定义如下，可以在前面一个宏定义里增加更多节约功耗的操作，如关闭外设，切换时钟，关闭电源等；也可以设置xExpectedIdleTime为0，不走接下来的进低功耗的流程，而自定义低功耗流程。后面一个宏定义是从低功耗退出后的处理，与前面的宏对应

* configPRE_SLEEP_PROCESSING(xExpectedIdleTime)
* configPOST_SLEEP_PROCESSING(xExpectedIdleTime)

下面是一些配置示例总结：

* 默认的低功耗机制，Systick定时器，频率与CPU Core同频，Tickless模式，设置宏USE_TICKLESS_IDLE=1
* Systick定时器，频率与CPU Core不同频，Tickless模式，设置USE_TICKLESS_IDLE=1，设置configSYSTICK_CLOCK_HZ为实际时钟频率
* 不使用Systick，而使用其他定时器产生RTOS的Tick中断，设置宏USE_TICKLESS_IDLE=2，自定义新定时器初始化函数vPortSetupTimerInterrupt()，设置宏configTICK_RATE_HZ，定义相应的xPortSysTickHandler()，新定时器的中断处理函数，同时注意CMSIS命名，不要与SysTick_Handler()产生冲突
* 使用现有的机制，只是将Systick替换成其他的定时器，目前FreeRTOS还没有相关的配置

FreeRTOS的低功耗机制已经基本学习完毕，结合ARM Cortex-M的低功耗，可以看出FreeRTOS的低功耗使用的是ARM的Sleep模式，通过STM32F4xx的[参考手册](http://www.stmcu.org/document/detail/index/id-200614)(Cortex-M4)，可以了解到其提供了3种低功耗模式，详见图示：

![stm32_CortexM4_low_power_intro]({{ "/images/stm32f4_cortexm4_low_power_intro.jpg" | prepend:site.baseurl }})

FreeRTOS里默认使用的是Sleep模式，还有2种分别是Stop模式和Standby模式。Standby模式是深度睡眠模式，最省电，但是一旦进入到这种低功耗模式，除了备份域(RTC与备份RAM)和待机电路寄存器外，寄存器和RAM中的内容全部Reset，基本可以理解为唤醒系统后软件从头运行，这种模式虽然省电，但是不适合应用在RTOS里面。再看一下Stop模式，进入这种模式的低功耗，1.2V电压域中的时钟全部停止，PLL/HSI/HSE RC振荡器也被禁止，内部SRAM和寄存器的内容会全部保留，但是在被唤醒之后，会有一个延迟，需要恢复时钟/恢复Flash的时间，如果使用调压器调压，也需要时间，这种模式也可以应用到RTOS里，但是，如果系统对实时性要求比较高，这样的省电模式就得慎重考虑如何使用了，还需要注意的一个问题是，当进入这种模式的低功耗之后，需要有一个在此时运行的定时器继续运行，在系统醒来之后进行计时补偿，如RTC。关于这几种功耗模式的使用，可以去参考标准固件库或者CubeMx固件库里的Examples。

### Memory Management

FreeRTOS提供了5种堆内存分配管理文件，详见源码Source/Portable/MemMang下面的5个heap_x.c文件，每一种都代表了1种应用场景，开发者可以自己选择使用哪一种，或者替换成自定义的。

* heap_1.c

堆内存管理中最简单的一个，只能申请分配内存，一旦申请成功不能释放，不会产生内存碎片，特别适合对安全要求高的场景。

在整片的堆内存上按照顺序依次获得需要大小的内存，xNextFreeByte用来记录已经分配的堆内存大小，(configADJUSTED_HEAP_SIZE - xNextFreeByte)计算剩余的堆内存，这里还要注意一个问题就是地址对齐，提高MCU的内存访问效率，STM32F4xx(Cortex-M4)的宏定义portBYTE_ALIGNMENT=8，即8字节对齐。

* heap_2.c

相比heap_1可以释放分配的堆内存，但是不会进行内存碎片整理，如果在使用堆内存时，如task/queue等每次用到的内存size是相同的，而不是变化不可确定的，打个比方如创建task需要分配内存，而所有的task创建时分配的内存大小相同，其他如Queue等也是如此，就可以使用这种方案，否则建议使用heap_4.c

heap_1分配可以看成是对数组的操作，heap_2允许释放内存，就需要使用链表的数据结构，定义如下：

```c
typedef struct A_BLOCK_LINK
{
  struct A_BLOCK_LINK *pxNextFreeBlock; /*<< The next free block in the list. */
  size_t xBlockSize; /*<< The size of the free block. */
} BlockLink_t;
```

维护了一个可分配堆内存块的单向链表，而且是按照可分配堆内存的大小升序排列的，初始化时xStart是链表头，指向第一个可分配堆内存块，xEnd用作链表尾：

```
升序单向链表：
xStart -> Free_Node_1 ... -> Free_Node_n -> xEnd

升序排列条件：(Free_Node_1.xBlockSize < Free_Node_n.xBlockSize)
xStart与xEnd为单独定义的链表节点，Free_Node_x都是堆内存里存储的节点

每个内存块在堆内存中的存储格式：
|------------------|
|BlockLink_t       |
| ->pxNextFreeBlock|
| ->xBlockSize     |
|------------------|
|                  |
|                  |
| 对应xBlockSize的  |
| 可分配的内存区     |
|                  |
|                  |
|------------------|

```

Malloc()时从链表头开始寻找，找到第一个满足size的Free堆内存块，将此内存块从这个可分配的堆内存块的链表中移除，如果满足条件的内存块较大，则分裂成2个，前一个正好满足需求的大小，剩余的全部分给新的内存块，然后将这个内存块插入到可分配堆内存块的链表里。Free()时直接将内存块插入到链表里。使用xFreeBytesRemaining保存剩余的可分配的内存大小。

* heap_3.c

封装了标准C库里的malloc()和free()，需要在编译器里指定堆内存，宏定义configTOTAL_HEAP_SIZE对其不起作用。

* heap_4.c

Malloc()时使用first fit算法，不像heap_2那样按照大小升序排列后找到size刚好匹配的那个，heap_4是按照地址前后排列的，malloc()时找到第一个合适大小的内存就分配，malloc()和free()中将相邻的Free的内存块合并成更大的Free内存块，更适合不确定大小的内存分配使用场景。

数据结构还是单链表，其定义与heap_2相同，还是维护一个可分配堆内存的链表，但不是升序排列，而是按地址前后顺序。初始化后xStart还是链表头，与之前相同都是指向第一个可分配的堆内存块，xEnd还是链表尾，但是存储在堆内存尾部，xFreeBytesRemaining还是保存剩余的可分配的内存大小，新增了2个变量，xMinimumEverFreeBytesRemaining用于保存当前可分配的堆内存块的最小size，xBlockAllocatedBit用于标记数值的最高位为1，如32bit的MCU，其值为0x80000000，在算法中与BlockLink_t->xBlockSize进行与or或操作，用于标示此内存块是否是Free可以分配的。

Malloc()还是遍历链表，找到第一个大小合适的内存就分配，如果此内存块更大，则分裂成两个节点，然后将Free的那个节点插入到可分配堆内存链表里，此链表节点按照地址先后顺序排列，每次寻找到合适的地址插入点，判断是否可以与前后节点合并成一个节点，然后进行合并或者插入链表，然后更新xFreeBytesRemaining、xFreeBytesRemaining，xBlockAllocatedBit操作。Free()也是完成上面的插入链表的操作。内存块合并操作在插入链表的过程中完成，相邻地址合并原则。

* heap_5.c

同样使用了heap_4中的first fit和内存块合并算法，不同的是heap_5应对的是堆内存分布在不连续的区域上的情况，如：

```c
/* Used by heap_5.c. */
typedef struct HeapRegion
{
  uint8_t *pucStartAddress;
  size_t xSizeInBytes;
} HeapRegion_t;

const HeapRegion_t xHeapRegions[] =
{
  { ( uint8_t * ) 0x80000000UL, 0x10000 },
  { ( uint8_t * ) 0x90000000UL, 0xa0000 },
  { NULL, 0 } /* Terminates the array. */
};
```

首先要对堆内存区域进行初始化，详见下面的初始化函数：

```c
void vPortDefineHeapRegions(const HeapRegion_t * const pxHeapRegions)
```

初始化函数就是将所有的堆内存块用链表链接起来，每一块地址连续的堆内存块格式如下：

```
|------------------|
|BlockLink_t       |
| ->pxNextFreeBlock|
| ->xBlockSize     |
|------------------|
|                  |
|                  |
| 对应xBlockSize的  |
| 可分配的内存区     |
|                  |
|                  |
|------------------|
|BlockLink_t       |
| ->pxNextFreeBlock|
| ->xBlockSize=0   |
|------------------|
```

然后将所有的堆内存块链接起来，相应的变量初始化好后，初始化工作就完成了。剩下的就与heap_4的使用完全相同了，完全看成是待分配的Free堆内存链表，只是不连续的区域可以认为是已经被分配的内存(永远不会被释放回来)，其Malloc()/Free()以及相应的链表插入操作与heap_4完全相同。

### Stack Usage and Stack Overflow Checking

Task Stack检查功能，通过宏定义configCHECK_FOR_STACK_OVERFLOW开启栈溢出检查功能，在vTaskSwitchContext()时做栈溢出检查，FreeRTOS提供了2种栈溢出检查方法，开发者需要自定义hook函数vApplicationStackOverflowHook()，栈溢出后再此函数内添加开发者自己的调试信息，此功能在开发和测试阶段使用。

* configCHECK_FOR_STACK_OVERFLOW=1

此方法检查栈地址是否越界来判断是否栈溢出，此方法速度快，但是在task运行时已经栈溢出，出栈后栈指针又回到正常范围内，就无法检测到了。

* configCHECK_FOR_STACK_OVERFLOW=2 (目前只要大于1)

此方法在task初始化时将整个栈初始化为特定值0xa5a5a5a5，检查最后16个byte是否被修改成其他值的方法来判断是否发生栈溢出，相比第一种方法是慢一些，但是确实避免了上面检测不到的情况。

关于Stack Overflow & Detecting的方法还有很多，还可以参看[uC/OS的一些方法总结](https://www.micrium.com/detecting-stack-overflows-part-2-of-2/)。使用MCU特定的栈指针地址保护(访问到某段地址就会产生Exception中断)，或者利用Cortex-M的MPU功能做地址保护，访问到被保护的地址产生Exception中断，或者利用软中断等方法。这样做的好处是在Task运行时就能够判断出是否栈溢出了，一旦发生栈溢出会产生Exception中断。

### Others

FreeRTOS还提供了很多有用的功能、Demo和资料:

* [Trace Features](http://www.freertos.org/rtos-trace-macros.html)，在RTOS里很多关键的位置预留了trace调试接口
* [Run Time Statistics](http://www.freertos.org/rtos-run-time-stats.html)，配置一个比RTOS的Tick定时器快10~100倍的定时器，调用接口得到如Tasks Run Time百分比统计信息等
* [Porting Guide](http://www.freertos.org/FreeRTOS-porting-guide.html)移植工作指导
* [Windows Simulator](http://www.freertos.org/FreeRTOS-Windows-Simulator-Emulator-for-Visual-Studio-and-Eclipse-MingW.html)/[Posix/Linux Simulator](http://www.freertos.org/FreeRTOS-simulator-for-Linux.html)模拟器相关
* [How FreeRTOS Works](http://www.freertos.org/implementation/main.html)基于AVR平台由浅入深的介绍FreeRTOS工作原理
* [Memory Protection Support](http://www.freertos.org/FreeRTOS-MPU-memory-protection-unit.html)MPU支持
* [API](http://www.freertos.org/a00106.html)详细介绍
* [Supported Devices & Demos](http://www.freertos.org/a00090.html)列出了FreeRTOS目前支持的芯片厂家及其产品系列
* [Demo Projects](http://www.freertos.org/a00102.html)列出了一些典型的Demo工程


## 小结

主要以FreeRTOS官网提供的资料为主，结合ARM Cortex-M，对一些主要功能进行了深入的理解，没有对FreeRTOS的API使用和构建应用进一步的深入，从应用层角度使用RTOS是基本类似的，也是比较容易上手的部分，需要的时候直接查看官方文档。