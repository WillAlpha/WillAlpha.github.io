---
layout: post
title: "RTOS学习笔记-FreeRTOS源码"
categories: RTOS
tags: ARM RTOS FreeRTOS Cortex-M
author: Will
---

* content
{:toc}


通过FreeRTOS官网资料，已经可以很好的使用FreeRTOS了，再深入的理解就需要深入到源码层，接下来就阅读源码，FreeRTOS版本v9.0.0，平台为ARM Cortex-M4，编译工具为ARM MDK。

FreeRTOS需要关注的源码：

```
//核心代码
list.c
task.c
queue.c

//平台相关，STM32(ARM Cortex-M4)
port.c
//已经分析过，Memory Management相关
heap_x.c

//非核心代码
event_groups.c
timers.c
croutine.c
```

## List

FreeRTOS核心数据结构List，源码查看list.c，List以及ListItem数据结构如下：

```c
/*
 * Definition of the only type of object that a list can contain.
 */
struct xLIST_ITEM
{
  listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE
  configLIST_VOLATILE TickType_t xItemValue;
  struct xLIST_ITEM * configLIST_VOLATILE pxNext;
  struct xLIST_ITEM * configLIST_VOLATILE pxPrevious;
  void * pvOwner;
  void * configLIST_VOLATILE pvContainer;
  listSECOND_LIST_ITEM_INTEGRITY_CHECK_VALUE
};
typedef struct xLIST_ITEM ListItem_t; /* For some reason lint wants this as two separate definitions. */

/*
 * Definition of the type of queue used by the scheduler.
 */
typedef struct xLIST
{
  listFIRST_LIST_INTEGRITY_CHECK_VALUE
  configLIST_VOLATILE UBaseType_t uxNumberOfItems;
  ListItem_t * configLIST_VOLATILE pxIndex;
  MiniListItem_t xListEnd;
  listSECOND_LIST_INTEGRITY_CHECK_VALUE
} List_t;
```

通过list.c源码的阅读，可以看出这是一个双向链表结构，图中的`List`表示这个双向链表，而`ListItem`就是双向链表里的节点，这些节点根据xItemValue的值的大小做了排序，如下图所示：

![FreeRTOS_list_model]({{ "/images/freertos_list_model.png" | prepend:site.baseurl }})

继续阅读源码，就能更深入的理解为什么这么定义数据结构，以及实际是如何使用的。

## Tasks

task创建函数xTaskCreate()

```c
BaseType_t xTaskCreate(...)
{
  //任务控制块TCB
  TCB_t *pxNewTCB;

  ...

  //分配堆内存给newTask的TCB和stack，注意portSTACK_GROWTH
  //初始化stack
  pxNewTCB->pxStack = pxStack;

  ...

  //newTask初始化
  prvInitialiseNewTask(...);
  //将task的TCB加入到Ready Tasks List
  prvAddNewTaskToReadyList(pxNewTCB);

  ...
}
```

这里有一个数据结构`TCB`，这个数据结构里面定义了所有与task相关的成员，数据结构比较大，暂时先不做过多说明，继续阅读源码，在接下来的源码中剖析`TCB`成员，以及前面的`List`如何使用。

```c
static void prvInitialiseNewTask(...)
{
  ...

  //如果启用Stack Overflow检查
  //则将stack初始化为特定值tskSTACK_FILL_BYTE
  ( void ) memset( pxNewTCB->pxStack, ( int ) tskSTACK_FILL_BYTE, ( size_t ) ulStackDepth * sizeof( StackType_t ) );

  ...

  //局部变量，先计算出stack的栈顶指针 pxTopOfStack
  pxTopOfStack = pxNewTCB->pxStack + ( ulStackDepth - ( uint32_t ) 1 );
  //地址对齐操作，stack操作更快
  pxTopOfStack = ( StackType_t * ) ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack ) & ( ~( ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) );

  ...

  //保存pxNewTCB->pcTaskName

  ...

  //保存task优先级
  pxNewTCB->uxPriority = uxPriority;
  #if ( configUSE_MUTEXES == 1 )
  {
    //Mutex有优先级继承，这里保存task原始的优先级
    pxNewTCB->uxBasePriority = uxPriority;
    pxNewTCB->uxMutexesHeld = 0;
  }
  #endif /* configUSE_MUTEXES */

  //ListItem_t初始化，状态与事件ListItem
  //Task在Ready/Blocked/Suspended某个状态时，此ListItem_t会挂到相应的List
  vListInitialiseItem( &( pxNewTCB->xStateListItem ) );
  //task相关的某个Event List会指向这个ListItem_t，如Queue满了而阻塞，将其挂接到等待入队的List
  vListInitialiseItem( &( pxNewTCB->xEventListItem ) );

  //ListItem_t->pvOwener指向此task的TCB
  listSET_LIST_ITEM_OWNER( &( pxNewTCB->xStateListItem ), pxNewTCB );

  //设置ListItem_t->xItemValue为优先级取反
  //逻辑优先级是数字越大优先级越高，存储的时候取反，数字越小优先级越高
  //配合前面List数据结构, 按照xItemValue大小排序，那么双链表首节点都是优先级最大的节点
  listSET_LIST_ITEM_VALUE( &( pxNewTCB->xEventListItem ), ( TickType_t ) configMAX_PRIORITIES - ( TickType_t ) uxPriority );
  //ListItem_t->pvOwener指向此task的TCB
  listSET_LIST_ITEM_OWNER( &( pxNewTCB->xEventListItem ), pxNewTCB );

  ...

  //task notification相关初始化
  #if ( configUSE_TASK_NOTIFICATIONS == 1 )
  {
    pxNewTCB->ulNotifiedValue = 0;
    pxNewTCB->ucNotifyState = taskNOT_WAITING_NOTIFICATION;
  }
  #endif

  ...

  //初始化stack，详见此函数解读，可了解stack保存内容及顺序
  pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters );

  ...
}

static void prvAddNewTaskToReadyList( TCB_t *pxNewTCB )
{
  //进入临界区
  taskENTER_CRITICAL();
  {
    uxCurrentNumberOfTasks++;
    if( pxCurrentTCB == NULL )
    {
      //没有任务或者其他任务都在Suspend状态
      //全局变量pxCurrentTCB指向此task TCB，永远指向当前运行的task TCB
      pxCurrentTCB = pxNewTCB;

      if( uxCurrentNumberOfTasks == ( UBaseType_t ) 1 )
      {
        //第一个任务，初始化Task List，详见对此函数的解读
        prvInitialiseTaskLists();
      }
    }
    else
    {
      //如果task调度还未开始，pxCurrentTCB指向优先级最大的那个task TCB
      if( xSchedulerRunning == pdFALSE )
      {
        if( pxCurrentTCB->uxPriority <= pxNewTCB->uxPriority )
        {
          pxCurrentTCB = pxNewTCB;
        }
      }
    }

    uxTaskNumber++;
    #if ( configUSE_TRACE_FACILITY == 1 )
    {
      /* Add a counter into the TCB for tracing only. */
      pxNewTCB->uxTCBNumber = uxTaskNumber;
    }
    #endif /* configUSE_TRACE_FACILITY */

    //将此task TCB插入到pxReadyTasksLists相应优先级List尾部
    prvAddTaskToReadyList( pxNewTCB );

    ...
  }
  //退出临界区
  taskEXIT_CRITICAL();

  //如果此时task开始调度，如果新task优先级更高，则立即发起一次调度
  if( xSchedulerRunning != pdFALSE )
  {
    if( pxCurrentTCB->uxPriority < pxNewTCB->uxPriority )
    {
      //强制产生一次调度，发起PendSV中断
      taskYIELD_IF_USING_PREEMPTION();
    }
  }
}

StackType_t *pxPortInitialiseStack(StackType_t *pxTopOfStack, TaskFunction_t pxCode, void *pvParameters)
{
  pxTopOfStack--;
  //状态寄存器入栈
  *pxTopOfStack = portINITIAL_XPSR;	/* xPSR */
  pxTopOfStack--;
  //PC指针入栈
  *pxTopOfStack = ( ( StackType_t ) pxCode ) & portSTART_ADDRESS_MASK;	/* PC */
  pxTopOfStack--;
  //链接返回寄存器入栈，保存返回地址
  *pxTopOfStack = ( StackType_t ) prvTaskExitError;	/* LR */

  /* Save code space by skipping register initialisation. */
  pxTopOfStack -= 5;	/* R12, R3, R2 and R1. */
  *pxTopOfStack = ( StackType_t ) pvParameters;	/* R0 */

  /* A save method is being used that requires each task to maintain its own exec return value. */
  pxTopOfStack--;
  *pxTopOfStack = portINITIAL_EXEC_RETURN; //返回值

  pxTopOfStack -= 8;	/* R11, R10, R9, R8, R7, R6, R5 and R4. */

  return pxTopOfStack;
}

static void prvInitialiseTaskLists( void )
{
  UBaseType_t uxPriority;

  //每个优先级创建一个Ready List，组成Ready List数组
  for( uxPriority = ( UBaseType_t ) 0U; uxPriority < ( UBaseType_t ) configMAX_PRIORITIES; uxPriority++ )
  {
    vListInitialise( &( pxReadyTasksLists[ uxPriority ] ) );
  }

  //创建Delay Task List
  vListInitialise( &xDelayedTaskList1 );
  vListInitialise( &xDelayedTaskList2 );
  //创建Pending Ready List
  vListInitialise( &xPendingReadyList );

  #if ( INCLUDE_vTaskDelete == 1 )
  {
    vListInitialise( &xTasksWaitingTermination );
  }
  #endif /* INCLUDE_vTaskDelete */

  #if ( INCLUDE_vTaskSuspend == 1 )
  {
    //创建Suspend Task List
    vListInitialise( &xSuspendedTaskList );
  }
  #endif /* INCLUDE_vTaskSuspend */

  /* Start with pxDelayedTaskList using list1 and the pxOverflowDelayedTaskList using list2. */
  pxDelayedTaskList = &xDelayedTaskList1;
  pxOverflowDelayedTaskList = &xDelayedTaskList2;
}
```

Task创建的过程已经基本清楚了，接着再看task调度过程，通过函数vTaskStartScheduler()启动调度：

```c
void vTaskStartScheduler( void )
{
  //先创建Idle Task
  xReturn = xTaskCreate(prvIdleTask, ...
  ...
  //如果用到Timers，还要创建Timers的task
  #if ( configUSE_TIMERS == 1 )
  {
    if( xReturn == pdPASS )
    {
      xReturn = xTimerCreateTimerTask();
    }
  }
  #endif /* configUSE_TIMERS */

  if( xReturn == pdPASS )
  {
    portDISABLE_INTERRUPTS();

    xNextTaskUnblockTime = portMAX_DELAY;
    //调度开始标志置位
    xSchedulerRunning = pdTRUE;
    //RTOS Tick初始化为0
    xTickCount = ( TickType_t ) 0U;

    //设置systick产生RTOS需要的Tick中断，与平台移植相关，详见分析
    xPortStartScheduler();
    ...
  }

  ...
}

BaseType_t xPortStartScheduler( void )
{
  ...

  //将PendSV和SysTick中断优先级设置为最低，与RTOS运行优先级相同
  portNVIC_SYSPRI2_REG |= portNVIC_PENDSV_PRI;
  portNVIC_SYSPRI2_REG |= portNVIC_SYSTICK_PRI;

  //启动Systick
  vPortSetupTimerInterrupt();

  //初始化嵌套计数为0
  uxCriticalNesting = 0;

  //浮点运算协处理器相关配置
  prvEnableVFP();
  *( portFPCCR ) |= portASPEN_AND_LSPEN_BITS;

  //启动第一个task
  prvStartFirstTask();

  //不会执行到这里！！！
  return 0;
}

void vPortSetupTimerInterrupt( void )
{
  #if configUSE_TICKLESS_IDLE == 1
  {
    //计算1Tick=多少Systick时钟频率计数
    ulTimerCountsForOneTick = ( configSYSTICK_CLOCK_HZ / configTICK_RATE_HZ );
    //低功耗每次关闭Systick的最大Tick数，更好的理解需要看前面Tickless模式低功耗的解读
    xMaximumPossibleSuppressedTicks = portMAX_24_BIT_NUMBER / ulTimerCountsForOneTick;
    //Cpu Cycles的补偿值，还是应用在Tickless模式低功耗
    ulStoppedTimerCompensation = portMISSED_COUNTS_FACTOR / ( configCPU_CLOCK_HZ / configSYSTICK_CLOCK_HZ );
  }
  #endif /* configUSE_TICKLESS_IDLE */

  //Systick tick中断加载值及使能，期望的频率是由宏configTICK_RATE_HZ定义(1ms中断)
  portNVIC_SYSTICK_LOAD_REG = ( configSYSTICK_CLOCK_HZ / configTICK_RATE_HZ ) - 1UL;
  portNVIC_SYSTICK_CTRL_REG = ( portNVIC_SYSTICK_CLK_BIT | portNVIC_SYSTICK_INT_BIT | portNVIC_SYSTICK_ENABLE_BIT );
}

__asm void prvStartFirstTask( void )
{
  PRESERVE8

  /* Use the NVIC offset register to locate the stack. */
  //0xE000ED08是Cortex-M4向量表偏移量寄存器(VTOR)的地址
  //起始地址存储着MSP即主堆栈指针
  ldr r0, =0xE000ED08
  ldr r0, [r0]
  ldr r0, [r0]
  /* Set the msp back to the start of the stack. */
  msr msp, r0
  /* Globally enable interrupts. */
  //使能全局中断
  cpsie i
  cpsie f
  dsb
  isb
  /* Call SVC to start the first task. */
  svc 0  //触发SVC中断，SVC中断处理函数启动第一个task
  nop
  nop
}

//SVC中断处理函数
__asm void vPortSVCHandler( void )
{
  PRESERVE8

  ldr r3, =pxCurrentTCB  //将pxCurrentTCB的值作为地址赋值给r3
  ldr r1, [r3]  //将pxCurrentTCB指针指向的值，当前task TCB的地址赋值给r1
  ldr r0, [r1]  //取得当前要运行的task栈顶指针，并赋值给r0，由此理解为什么TCB数据结构的第一项必须设计为栈顶指针
  ldmia r0!, {r4-r11}  //寄存器r4~r11出栈
  msr psp, r0  //栈顶指针赋给线程堆栈指针PSP
  isb
  mov r0, #0
  msr basepri, r0
  bx r14  //跳转执行目标task
}
```

三个特殊的中断及中断处理函数，Systick/SVC/PendSV，这也是在移植FreeRTOS时需要特别注意的地方，Systick产生RTOS需要的Tick中断，其中断处理函数与RTOS密切相关，SVC如上所示启动Task，PendSV用于task调度切换，Systick/PendSV配置成了最低优先级，在中断抢占的前提下，PendSV被抢占但是在处理完高优先级的任务后，依然会进入中断处理函数处理，这样不会打断高优先级的任务，也能完成任务的调度切换。

Task调度机制启动后，任务就开始执行，FreeRTOS里面的任务切换一般在PendSV中断处理函数里面进行，而PendSV中断则是由Systick中断中触发的，也就是系统Tick中断中判断是否有任务需要切换，需要则产生PendSV中断，然后再PendSV中断处理函数里面执行切换操作。另外，在xCreateTask()过程分析中(Idle Task里面也有这样的操作)，我们也见到了直接切换的方式taskYIELD()，产生PendSV中断。总结2种方式：

```c
//第一种，直接产生PendSV中断
portYIELD或者portYIELD_FROM_ISR

#define portYIELD()
{
  portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;			 

  __dsb( portSY_FULL_READ_WRITE );
  __isb( portSY_FULL_READ_WRITE );
}

//第二种，判断是否有task需要切换，然后再决定是否产生PendSV中断
void xPortSysTickHandler( void )
{
  vPortRaiseBASEPRI();
  {
    if(xTaskIncrementTick() != pdFALSE)
    {
      portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
    }
  }
  vPortClearBASEPRIFromISR();
}
```

具体的切换过程即为PendSV的中断处理函数，汇编+C实现，还是有一些ARM Cortex-M4相关的部分放置在port.c中，而且这部分C语言也很难实现。

```
__asm void xPortPendSVHandler( void )
{
  extern uxCriticalNesting;
  extern pxCurrentTCB;
  extern vTaskSwitchContext;

  PRESERVE8

  //先进行当前task的入栈操作，task的栈指针寄存器使用psp
  //中断服务程序处理之前，自动入栈xPSR、PC、LR、R12、R3~R0
  mrs r0, psp
  isb
  /* Get the location of the current TCB. */
  ldr	r3, =pxCurrentTCB
  ldr	r2, [r3]

  /* Is the task using the FPU context?  If so, push high vfp registers. */
  //FPU入栈
  tst r14, #0x10
  it eq
  vstmdbeq r0!, {s16-s31}

  /* Save the core registers. */
  //其他寄存器入栈
  stmdb r0!, {r4-r11, r14}

  /* Save the new top of stack into the first member of the TCB. */
  //更新最新的栈指针到当前task TCB首地址(即第一项保存当前栈指针)
  str r0, [r2]

  //R3入栈，后面调用vTaskSwitchContext()，此函数从ReadyList里取出即将运行的task TCB赋值给pxCurrentTCB
  //R3保存了pxCurrentTCB地址
  stmdb sp!, {r3}
  mov r0, #configMAX_SYSCALL_INTERRUPT_PRIORITY
  msr basepri, r0
  dsb
  isb
  bl vTaskSwitchContext
  mov r0, #0
  msr basepri, r0
  ldmia sp!, {r3}
  //恢复R3，此时的pxCurrentTCB已经指向了即将运行的task TCB

  /* The first item in pxCurrentTCB is the task top of stack. */
  //取得task TCB的栈指针
  ldr r1, [r3]
  ldr r0, [r1]

  /* Pop the core registers. */
  //部分寄存器出栈
  ldmia r0!, {r4-r11, r14}

  /* Is the task using the FPU context?  If so, pop the high vfp registers too. */
  //FPU出栈
  tst r14, #0x10
  it eq
  vldmiaeq r0!, {s16-s31}

  //将task的当前栈指针赋值给psp
  msr psp, r0
  isb
  bx r14 //跳转至即将运行的task运行，R0~R3、R12、LR、PC、xPSR自动出栈
}
```

Cortex-M4提供了2个栈指针MSP和PSP，PSP就是MCU正常运行时使用的栈指针，因此一般PSP都指向了某个task的栈，而MSP在异常情况下使用，MSP一般指向的是整个系统的栈，两个栈指针寄存器分工不同。通过上面的过程，也可以看到出栈入栈的顺序：

```
	|             | Stack
	|-------------|
	|    xPSR     |
	|    PC       |
	|    LR       |
	|    R12      |
	|    R3       |
	|    ...      |
	|    R0       |
	|    R14      |
	|    R11      |
	|    ...      |
	|    R4       | <-PSP
	|-------------|
	|             |
```

Task相关的创建、调度及切换的源码基本读了一遍，还有些细节暂放一下，继续阅读。

## Queue

队列应用于任务间通讯，可以在任务与任务之间，中断服务程序与任务之间传递消息，消息是通过Copy进队列的方式传递的，队列维护消息体本身，多了一次Copy而不是使用引用，对消息体本身的安全性和完整性有益。另外，Semaphore/Mutex也是借助Queue实现的，需要对Queue进一步的理解。

队列的数据结构定义为xQUEUE，此结构较大，暂时不对其细致的理解，后面阅读源码过程中深入理解，再看队列创建函数xQueueCreate()，实际是xQueueGenericCreate()函数。

```c
QueueHandle_t xQueueGenericCreate(...)
{
  //Queue分配堆内存  

  //初始化Queue
  prvInitialiseNewQueue(...);
}
```

为Queue分配堆内存大小及顺序：Queue_t结构体+队列消息内容占用内存总和。

```
	|-----------------------|
	|    Queue_t结构体      |
	|-----------------------|
	|    Queue所有队列项     |
	|-----------------------|
```

再看具体的Queue初始化源码：

```c
static void prvInitialiseNewQueue(...)
{
  ...

  //初始化完善Queue_t数据结构的各项
  if( uxItemSize == ( UBaseType_t ) 0 )
  {
    //Mutex时没有队列消息项，直接指向Queue_t首地址
    pxNewQueue->pcHead = ( int8_t * ) pxNewQueue;
  }
  else
  {
    //队列头指向队列消息项的首地址
    pxNewQueue->pcHead = ( int8_t * ) pucQueueStorage;
  }

  pxNewQueue->uxLength = uxQueueLength;
  pxNewQueue->uxItemSize = uxItemSize;
  //更多的初始化，继续深入阅读
  ( void ) xQueueGenericReset(pxNewQueue, pdTRUE);
  ...
}

BaseType_t xQueueGenericReset(QueueHandle_t xQueue, BaseType_t xNewQueue)
{
  ...

  taskENTER_CRITICAL();
  {
    //tail指向队列尾
    pxQueue->pcTail = pxQueue->pcHead + ( pxQueue->uxLength * pxQueue->uxItemSize );
    //当前队列里的队列项个数
    pxQueue->uxMessagesWaiting = ( UBaseType_t ) 0U;
    //指向下一个可写的队列项，用于队列入队时向队尾添加队列项
    pxQueue->pcWriteTo = pxQueue->pcHead;
    //指向最后一个队列项，用于队列入队时向队首添加队列项
    pxQueue->u.pcReadFrom = pxQueue->pcHead + ( ( pxQueue->uxLength - ( UBaseType_t ) 1U ) * pxQueue->uxItemSize );
    //初始化Queue Lock时接收和发送的队列项
    pxQueue->cRxLock = queueUNLOCKED;
    pxQueue->cTxLock = queueUNLOCKED;

    ...

    //初始化了2个List，记录了等待超时和接收消息的tasks List
    vListInitialise( &( pxQueue->xTasksWaitingToSend ) );
    vListInitialise( &( pxQueue->xTasksWaitingToReceive ) );
  }
  taskEXIT_CRITICAL();

  return pdPASS;
}
```

初始化完成后的Queue模型如下图所示：

![FreeRTOS_queue_model]({{ "/images/freertos_queue_model.png" | prepend:site.baseurl }})

前面是Queue_t结构，后面紧跟着的Item_x为实际队列项(队列消息体)，Queue_t中的两个List存储的是与此队列相关联的tasks。继续看xQueueSend()也就是xQueueGenericSend()源码

```c
BaseType_t xQueueGenericSend(...）
{
  ...

  for(;;)
  {
    taskENTER_CRITICAL();
    {
      //检查队列是否满，如果是overwrite方式直接入队
      if( ( pxQueue->uxMessagesWaiting < pxQueue->uxLength ) || ( xCopyPosition == queueOVERWRITE ) )
      {
        //三种入队方式：队尾，队首和overwrite
        //特别注意当用作Mutex使用时，返回值可能为True，即需要切换到更高优先级任务，需要执行任务调度
        xYieldRequired = prvCopyDataToQueue( pxQueue, pvItemToQueue, xCopyPosition );

        ...

        //先忽略掉Queue Sets的情况
        //如果有tasks正在等待接收队列消息
        if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToReceive ) ) == pdFALSE )
        {
          //如果有任务在等待队列消息，则将此任务添加进任务Ready List
          if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToReceive ) ) != pdFALSE )
          {
            queueYIELD_IF_USING_PREEMPTION();
          }
        }
        else if (xYieldRequired != pdFALSE)
        {
          //仅用作Mutex时，释放Mutex后调度任务(Mutex有优先级继承)
          queueYIELD_IF_USING_PREEMPTION();
        }
        else
        {
        }
        taskEXIT_CRITICAL();
        return pdPASS;
      }
      else //队列满，且非overwrite方式入队
      {
        if( xTicksToWait == ( TickType_t ) 0 )
        {
          //队列满了，而且没有设置超时等待，则直接退出，返回队列满错误
          taskEXIT_CRITICAL();
          return errQUEUE_FULL;
        }
        else if( xEntryTimeSet == pdFALSE )
        {
          //队列满了，而且设置了超时等待，则初始化了一个timeout数据结构
          vTaskSetTimeOutState( &xTimeOut );
          xEntryTimeSet = pdTRUE;
        }
        else
        {
        }
      }      
    }
    taskEXIT_CRITICAL();
    //退出临界区，此处可能会有任务调度

    //全部任务挂起，阻止任务调度
    vTaskSuspendAll();
    //锁定Queue，将Queue_t里的rxLock和txLock设置为Lock，阻止了任务调度，但是中断处理程序里还是可以对Queue进行操作，因此对Queue上锁
    prvLockQueue( pxQueue );

    if( xTaskCheckForTimeOut( &xTimeOut, &xTicksToWait ) == pdFALSE )
    {
      //队列等待超时未过期
      if( prvIsQueueFull( pxQueue ) != pdFALSE )
      {
        //队列满
        //将当前task加入到等待此队列入队的等待List和延时List
        vTaskPlaceOnEventList( &( pxQueue->xTasksWaitingToSend ), xTicksToWait );
        //解锁队列
        prvUnlockQueue( pxQueue );
        //将tasks从PendingReadyList移至ReadyList
        if( xTaskResumeAll() == pdFALSE )
        {
          portYIELD_WITHIN_API();
        }
      }
      else
      {
        //队列未满
        //解锁队列，恢复挂起的所有任务，没有return，继续for循环
        prvUnlockQueue( pxQueue );
        ( void ) xTaskResumeAll();
      }
    }
    else
    {
      //设置的队列等待超时过期
      prvUnlockQueue( pxQueue );
      //挂起的tasks全部恢复，返回队列满错误
      ( void ) xTaskResumeAll();
      return errQUEUE_FULL;
    }
  }
}
```

队列入队后，如果队列未满，将等待此队列消息的List(xTasksWaitingToReceive)中需要解除阻塞的task从等待消息的List中删除，然后将其添加进任务Ready List中，然后再与当前运行的task的优先级做一下比较，如果优先级更高则产生一次调度。详见函数xTaskRemoveFromEventList()。

如果队列满了，则情况稍微复杂一些，如果队列入队时没有设置阻塞等待时间，即xTicksToWait=0，则直接返回队列满错误；如果设置了阻塞等待时间，而且时间未到，则将当前运行的task加入到等待此队列入队的等待List里(xTasksWaitingToSend)，还要将此任务加入到延时List中，可参考函数vTaskPlaceOnEventList()，解锁队列，恢复所有挂起的任务，恢复调度，如果此时有更高优先级的任务Ready，则产生一次任务调度；如果设置了阻塞时间，而且时间到，则解锁Queue，恢复所有挂起的任务，恢复调度，返回队列满错误。

接着再看一下中断处理函数里相关的Queue的操作函数xQueueGenericSendFromISR()

```c
BaseType_t xQueueGenericSendFromISR(...)
{
  ...
  uxSavedInterruptStatus = portSET_INTERRUPT_MASK_FROM_ISR();
  {
    if( ( pxQueue->uxMessagesWaiting < pxQueue->uxLength ) || ( xCopyPosition == queueOVERWRITE ) )
    {
      //队列未满或overwrite
      //入队，分三种方式：队尾、队首、overwrite
      ( void ) prvCopyDataToQueue( pxQueue, pvItemToQueue, xCopyPosition );

      if( cTxLock == queueUNLOCKED )
      {
        //如果队列unlock
        ...

        //队列非空
        if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToReceive ) ) == pdFALSE )
        {
          //如果有任务在等待队列消息，则将此任务添加进任务Ready List
          if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToReceive ) ) != pdFALSE )
          {
            //设置为pdTRUE，则代表需要一次任务切换
            if( pxHigherPriorityTaskWoken != NULL )
            {
              *pxHigherPriorityTaskWoken = pdTRUE;
            }
          }
        }
      }
      else
      {
        //如果队列lock，此时不可以操作队列，只有tx锁计数器加1
        pxQueue->cTxLock = ( int8_t ) ( cTxLock + 1 );
      }

      xReturn = pdPASS;
    }
    else
    {
      //队列满了，直接返回队列满错误
      xReturn = errQUEUE_FULL;
    }
  }
  portCLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus );

  return xReturn;
}
```

中断服务程序里的队列入队操作就要简单许多，不同的处理是在队列被锁定后，不再做任何操作了，直接将cTxLock计数器加1，在队列被解锁后，根据计数器的值，依次处理相关的队列项。

再接着看出队函数xQueueReceive()，也就是xQueueGenericReceive()，与入队的操作相反，基本逻辑非常类似。

```c
BaseType_t xQueueGenericReceive(...)
{
  ...
  for( ;; )
  {
    taskENTER_CRITICAL();
    {
      const UBaseType_t uxMessagesWaiting = pxQueue->uxMessagesWaiting;
      //等待消息队列项非空
      if( uxMessagesWaiting > ( UBaseType_t ) 0 )
      {
        pcOriginalReadPosition = pxQueue->u.pcReadFrom;

        //出队操作，不像前面入队那样分三种情况了，出队只有copy操作
        prvCopyDataFromQueue( pxQueue, pvBuffer );

        if( xJustPeeking == pdFALSE )
        {
          //需要移除队列消息项，一般情况都采用这种操作，走这个分支
          pxQueue->uxMessagesWaiting = uxMessagesWaiting - 1;

          #if ( configUSE_MUTEXES == 1 )
          {
            //Mutex时的操作
            if( pxQueue->uxQueueType == queueQUEUE_IS_MUTEX )
            {
              pxQueue->pxMutexHolder = ( int8_t * ) pvTaskIncrementMutexHeldCount();
            }
          }
          #endif

          if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToSend ) ) == pdFALSE )
          {
            //队列等待发送List非空，有task在等待向队列发送消息
            //将此任务添加进任务Ready List
            if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToSend ) ) != pdFALSE )
            {
              //如果有任务需要调度，产生一次任务调度
              queueYIELD_IF_USING_PREEMPTION();
            }
          }
        }
        else
        {
          //不需要移除队列消息项
          pxQueue->u.pcReadFrom = pcOriginalReadPosition;

          if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToReceive ) ) == pdFALSE )
          {
            if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToReceive ) ) != pdFALSE )
            {
              queueYIELD_IF_USING_PREEMPTION();
            }
          }
        }

        taskEXIT_CRITICAL();
        return pdPASS;
      }
      else
      {
        //等待消息队列项为空
        if( xTicksToWait == ( TickType_t ) 0 )
        {
          //设置等待阻塞超时时间为0，则直接返回队列空错误
          taskEXIT_CRITICAL();

          return errQUEUE_EMPTY;
        }
        else if( xEntryTimeSet == pdFALSE )
        {
          //设置了等待阻塞超时时间，则初始化一个TimeOut数据结构
          vTaskSetTimeOutState( &xTimeOut );
          xEntryTimeSet = pdTRUE;
        }
        else
        {
        }
      }
    }
    taskEXIT_CRITICAL();
    //退出临界区，此处可能会有任务调度

    //所有任务挂起，阻止任务调度
    vTaskSuspendAll();
    //Queue锁定，此时可以处理中断，中断处理程序里可能对队列有操作
    prvLockQueue( pxQueue );

    if( xTaskCheckForTimeOut( &xTimeOut, &xTicksToWait ) == pdFALSE )
    {
      //等待阻塞超时时间未到
      if( prvIsQueueEmpty( pxQueue ) != pdFALSE )
      {
        //队列未空
        #if ( configUSE_MUTEXES == 1 )
        {
          //Mutex时的操作
          if( pxQueue->uxQueueType == queueQUEUE_IS_MUTEX )
          {
            taskENTER_CRITICAL();
            {
              vTaskPriorityInherit( ( void * ) pxQueue->pxMutexHolder );
            }
            taskEXIT_CRITICAL();
          }
        }
        #endif

        //将任务加入到等待接收队列消息的List和延时List
        vTaskPlaceOnEventList( &( pxQueue->xTasksWaitingToReceive ), xTicksToWait );
        //Queue解锁
        prvUnlockQueue( pxQueue );
        //挂起的任务全部恢复，并启动调度
        if( xTaskResumeAll() == pdFALSE )
        {
          //如果有调度需要，产生一次调度
          portYIELD_WITHIN_API();
        }
      }
      esle
      {
        //队列为非空
        prvUnlockQueue( pxQueue );
        ( void ) xTaskResumeAll();
      }
    }
    else
    {
      //等待阻塞超时时间到
      //Queue解锁，恢复挂起的任务，开启任务调度
      prvUnlockQueue( pxQueue );
      ( void ) xTaskResumeAll();

      if( prvIsQueueEmpty( pxQueue ) != pdFALSE )
      {
        //队列未空，返回队列空错误
        return errQUEUE_EMPTY;
      }
    }
  }
}
```

另一个中断处理程序里的函数xQueueReceiveFromISR()也与相应的入队时类似。

```c
BaseType_t xQueueReceiveFromISR(...)
{
  uxSavedInterruptStatus = portSET_INTERRUPT_MASK_FROM_ISR();
  {
    const UBaseType_t uxMessagesWaiting = pxQueue->uxMessagesWaiting;
    if( uxMessagesWaiting > ( UBaseType_t ) 0 )
    {
      ...
      //将队列消息项copy出队列
      prvCopyDataFromQueue( pxQueue, pvBuffer );
      pxQueue->uxMessagesWaiting = uxMessagesWaiting - 1;

      if( cRxLock == queueUNLOCKED )
      {
        //队列unlock
        if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToSend ) ) == pdFALSE )
        {
          //有task等待向队列发送消息，将此task从等待发送队列的List中移除，并加入到任务的Ready List里
          if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToSend ) ) != pdFALSE )
          {
            if( pxHigherPriorityTaskWoken != NULL )
            {
              //有任务调度，将任务调度标志置位pdTRUE
              *pxHigherPriorityTaskWoken = pdTRUE;
            }
          }
        }
      }
      else
      {
        //队列lock，队列不进行操作，将rxLock计数器加1，后面队列解锁后再依次做相应的处理
        pxQueue->cRxLock = ( int8_t ) ( cRxLock + 1 );
      }
    }
    else
    {
      //队列里没有接收的消息，直接返回失败错误
      xReturn = pdFAIL;
    }
  }
  portCLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus );

  return xReturn;
}
```

## Semaphore & Mutex

信号量和互斥量也是基于Queue实现的，有了前面阅读Queue相关源码的基础，继续阅读这两者就能更好的理解。

Binary Semaphore创建函数xSemaphoreCreateBinary()，从下面的宏定义看出创建了一个type为queueQUEUE_TYPE_BINARY_SEMAPHORE，队列size为1，队列项size为0的队列。

```c
#define xSemaphoreCreateBinary()
xQueueGenericCreate( ( UBaseType_t ) 1, semSEMAPHORE_QUEUE_ITEM_LENGTH, queueQUEUE_TYPE_BINARY_SEMAPHORE )
```

Counting Semaphore创建函数xSemaphoreCreateCounting()，由下面的代码可以看出是创建一个type为queueQUEUE_TYPE_COUNTING_SEMAPHORE，队列size为创建时的参数uxMaxCount(计数最大值)，队列项size为0的队列。另一个初始化时的参数uxInitialCount是信号量的初始值(即为队列里消息项的个数)。

```c
#define xSemaphoreCreateCounting( uxMaxCount, uxInitialCount )
xQueueCreateCountingSemaphore( ( uxMaxCount ), ( uxInitialCount ) )

QueueHandle_t xQueueCreateCountingSemaphore(...)
{
  QueueHandle_t xHandle;

  xHandle = xQueueGenericCreate( uxMaxCount, queueSEMAPHORE_QUEUE_ITEM_LENGTH, queueQUEUE_TYPE_COUNTING_SEMAPHORE );

  if( xHandle != NULL )
  {
    ( ( Queue_t * ) xHandle )->uxMessagesWaiting = uxInitialCount;
  }

  return xHandle;
}
```

Mutex的创建函数xSemaphoreCreateMutex()，由下面的代码可以看出，创建了一个type为queueQUEUE_TYPE_MUTEX，队列size为1，队列项size为0的队列。

```c
#define xSemaphoreCreateMutex() xQueueCreateMutex( queueQUEUE_TYPE_MUTEX )

QueueHandle_t xQueueCreateMutex( const uint8_t ucQueueType )
{
  Queue_t *pxNewQueue;
  const UBaseType_t uxMutexLength = ( UBaseType_t ) 1, uxMutexSize = ( UBaseType_t ) 0;

  pxNewQueue = ( Queue_t * ) xQueueGenericCreate( uxMutexLength, uxMutexSize, ucQueueType );
  prvInitialiseMutex( pxNewQueue );

  return pxNewQueue;
}

static void prvInitialiseMutex( Queue_t *pxNewQueue )
{
  if( pxNewQueue != NULL )
  {
    //实际是在Mutex情况下，将Queue_t的pcTail和pcHead赋予了新的意义
    pxNewQueue->pxMutexHolder = NULL;
    pxNewQueue->uxQueueType = queueQUEUE_IS_MUTEX;

    //用于Recursive Mutex
    pxNewQueue->u.uxRecursiveCallCount = 0;

    //这里相当于先释放了Mutex，即使用Mutex第一次就可以获取资源，区别于Binary Semaphore
    ( void ) xQueueGenericSend( pxNewQueue, NULL, ( TickType_t ) 0U, queueSEND_TO_BACK );
  }
}
```

通过Mutex和Binary Semaphore的初始化源码可以看出，在使用上Mutex创建完成后，可以直接获得资源，然后用完了再释放；而Binary Semaphore不同，创建完成后不能直接获取到，需要先释放再获取。

Recursive Mutex的创建函数xSemaphoreCreateRecursiveMutex()，实际与Mutex初始化相同，只是一个类型的区别queueQUEUE_TYPE_RECURSIVE_MUTEX，默认FreeRTOS不打开此功能。

```c
#define xSemaphoreCreateRecursiveMutex() xQueueCreateMutex( queueQUEUE_TYPE_RECURSIVE_MUTEX )
```

接着看Semaphore和Mutex的Take和Give，其中Binary Semaphore/Semaphore/Mutex的API相同，Recursive Mutex有单独的API

```c
//Take操作就是Queue的出队操作
#define xSemaphoreTake( xSemaphore, xBlockTime )
xQueueGenericReceive( ( QueueHandle_t ) ( xSemaphore ), NULL, ( xBlockTime ), pdFALSE )

//特别注意Mutex不能在中断中使用，因此这个API只适用于Semaphore
#define xSemaphoreTakeFromISR( xSemaphore, pxHigherPriorityTaskWoken )
xQueueReceiveFromISR( ( QueueHandle_t ) ( xSemaphore ), NULL, ( pxHigherPriorityTaskWoken ) )

//Give操作就是Queue的入队操作
#define xSemaphoreGive(xSemaphore)
xQueueGenericSend( ( QueueHandle_t ) ( xSemaphore ), NULL, semGIVE_BLOCK_TIME, queueSEND_TO_BACK )

//特别注意Mutex不能在中断中使用，因此这个API只适用于Semaphore
#define xSemaphoreGiveFromISR( xSemaphore, pxHigherPriorityTaskWoken )
xQueueGiveFromISR( ( QueueHandle_t ) ( xSemaphore ), ( pxHigherPriorityTaskWoken ) )
```

Give操作就是Queue的入队操作，队列满了就返回满错误；未满则入队，计算加1，判断是否有任务阻塞，如果有则任务调度。Take操作就是Queue的出队操作，队列不为空，则计数减1，判断是否有任务入队阻塞，如果有则任务调度；队列为空，阻塞等待时间为0，则直接返回空错误；队列为空，阻塞等待时间不为0，则任务阻塞，并将任务加入延时列表。特别注意Mutex不能在中断处理函数中操作。

Recursive Mutex的Take和Give的API区别于其他3个

```c
#define xSemaphoreTakeRecursive( xMutex, xBlockTime )
xQueueTakeMutexRecursive( ( xMutex ), ( xBlockTime ) )

BaseType_t xQueueTakeMutexRecursive( QueueHandle_t xMutex, TickType_t xTicksToWait )
{
  BaseType_t xReturn;
  Queue_t * const pxMutex = ( Queue_t * ) xMutex;

  if( pxMutex->pxMutexHolder == ( void * ) xTaskGetCurrentTaskHandle() )
  {
    //非第一次调用此API，此时pxMutexHolder与当前task的TCB相同，直接uxRecursiveCallCount加1，不用去操作队列函数；如果此时是另一个任务Take，则会到下面的分支阻塞在xQueueGenericReceive
    //同一个任务只要递归Mutex没有将所有Take的次数Give掉，就直接进入这个分支；同一个任务一旦Give掉所有的Take次数，pxMutexHolder置为NULL，就会进入下面的分支，此时有其他任务想Take此Mutex会获得Take的机会
    ( pxMutex->u.uxRecursiveCallCount )++;
    xReturn = pdPASS;
  }
  else
  {
    //如果第一次调用此API去Take递归Mutex，将当前Task的TCB赋值给pxMutexHolder，然后将uxRecursiveCallCount加1
    //如果递归Mutex的Give所有Take的次数，将pxMutexHolder置为NULL，则又进入到这个分支，如果此时有其他任务Take递归Mutex，则会获得机会，否则只能被阻塞在xQueueGenericReceive
    xReturn = xQueueGenericReceive( pxMutex, NULL, xTicksToWait, pdFALSE );

    if( xReturn != pdFAIL )
    {
      ( pxMutex->u.uxRecursiveCallCount )++;
    }
  }

  return xReturn;
}

#define xSemaphoreGiveRecursive( xMutex )
xQueueGiveMutexRecursive( ( xMutex ) )

BaseType_t xQueueGiveMutexRecursive( QueueHandle_t xMutex )
{
  BaseType_t xReturn;
  Queue_t * const pxMutex = ( Queue_t * ) xMutex;

  //判断是否在同一个任务中Give
  if( pxMutex->pxMutexHolder == ( void * ) xTaskGetCurrentTaskHandle() )
  {
    //通过uxRecursiveCallCount计数递归，每次Give操作此值减1
    ( pxMutex->u.uxRecursiveCallCount )--;

    if( pxMutex->u.uxRecursiveCallCount == ( UBaseType_t ) 0 )
    {
      //如果Give的次数刚好与Take的次数相等，向pxMutex队列里发送一条消息，pxMutexHolder置为NULL，这样Give次数过多就不会再走进这个分支，而是直接返回错误；另外如果有其他任务需要Take递归Mutex，则获得机会，即释放了递归Mutex
      ( void ) xQueueGenericSend( pxMutex, NULL, queueMUTEX_GIVE_BLOCK_TIME, queueSEND_TO_BACK );
    }

    xReturn = pdPASS;
  }
  else
  {
    //如果不在同一个任务中直接返回错误，还有Give的次数超过了Take的次数也会走到这里
    xReturn = pdFAIL;
  }

  return xReturn;
}
```

由上面的源码可以看出，Mutex需要在同一个任务中获取释放，作为资源共享锁的使用方式，pxMutexHolder会记录创建Mutex时的task TCB，Take和Give时都会判断是否是在同一个task中，如果不是直接返回错误。而Binary Semaphore没有优先级继承，而且可以在任意的任务中获取释放。

Mutex具有优先级继承，主要用作资源共享时，提升当前获得Mutex的任务的优先级至等待此资源的所有任务中的最高优先级，尽最大可能的避免优先级翻转造成的危害(高优先级任务一直得不到资源一直被挂起，或者直接死锁了)。

可以看出，Semaphore和Mutex都是使用Queue实现的，只用到了Queue的头部分，即Queue_t结构体，而Queue的队列项则为空。Binary Semaphore/Semaphore/Mutex/Recursive Mutex各有自己的创建API，最终都是调用的Queue的创建函数；Binary Semaphore/Semaphore/Mutex的Take和Give操作API相同，Recursive Mutex有自己的单独的API操作；Semaphore有中断相关的API，但是Mutex不能在中断处理程序中执行，Mutex具有优先级继承，而且必须在同一个任务中Take和Give，而Semaphore没有优先级继承，可以在任意的任务中Take，然后在任意的任务中Give，或者反过来操作。

## Task Notifications

Task Notifications是FreeRTOS V8.2.0之后新增的功能，官方文档结论是比Queue/Semaphore/Mutex/Event Groups更快，使用RAM更少，可以携带长度为1的消息内容，完全基于Task实现。

Task TCB数据结构里相关的定义如下：

```c
typedef struct tskTaskControlBlock
{
  ...

#if( configUSE_TASK_NOTIFICATIONS == 1 )
  volatile uint32_t ulNotifiedValue;
  volatile uint8_t ucNotifyState;
#endif

  ...
} tskTCB;
```

Task Notifications的发送通知函数xTaskGenericNotify()

```c
typedef enum
{
  eNoAction = 0, //发送通知但不使用ulValue值，无通知内容
  eSetBits, //被通知的任务的通知值按bit或ulValue
  eIncrement, //被通知的任务的通知值加1
  eSetValueWithOverwrite, //被通知任务的通知值直接设置成ulValue，无论之前的通知值是否已经被读取
  eSetValueWithoutOverwrite	//之前的任务通知值被读取后，再更新被通知任务的通知值，如果没有读取则丢弃当前的ulValue
} eNotifyAction;

BaseType_t xTaskGenericNotify(...)
{
  TCB_t * pxTCB;
  BaseType_t xReturn = pdPASS;
  uint8_t ucOriginalNotifyState;

  pxTCB = ( TCB_t * ) xTaskToNotify;

  taskENTER_CRITICAL();
  {
    if( pulPreviousNotificationValue != NULL )
    {
      //函数传入的参数，将ulNotifiedValue更新前的值赋值给这个传入参数，发送通知的task获得更新前的通知值
      *pulPreviousNotificationValue = pxTCB->ulNotifiedValue;
    }

    //保存ucNotifyState
    ucOriginalNotifyState = pxTCB->ucNotifyState;
    //更新ucNotifyState
    pxTCB->ucNotifyState = taskNOTIFICATION_RECEIVED;

    //更新通知的方法，详见eNotifyAction枚举定义
    switch( eAction )
    {
      case eSetBits	:
        pxTCB->ulNotifiedValue |= ulValue;
        break;

      case eIncrement	:
        ( pxTCB->ulNotifiedValue )++;
        break;

      case eSetValueWithOverwrite	:
        pxTCB->ulNotifiedValue = ulValue;
        break;

      case eSetValueWithoutOverwrite :
        if( ucOriginalNotifyState != taskNOTIFICATION_RECEIVED )
        {
          pxTCB->ulNotifiedValue = ulValue;
        }
        else
        {
          xReturn = pdFAIL;
        }
        break;

      case eNoAction:
        break;
    }

    if( ucOriginalNotifyState == taskWAITING_NOTIFICATION )
    {
      //被通知的任务正好在等待通知的状态，则将其添加到任务Ready List
      ( void ) uxListRemove( &( pxTCB->xStateListItem ) );
      prvAddTaskToReadyList( pxTCB );

      #if( configUSE_TICKLESS_IDLE != 0 )
      {
        prvResetNextTaskUnblockTime();
      }
      #endif

      if( pxTCB->uxPriority > pxCurrentTCB->uxPriority )
      {
        //如果发现等待通知的任务优先级更高，则触发一次任务调度
        taskYIELD_IF_USING_PREEMPTION();
      }
  }
  taskEXIT_CRITICAL();

  return xReturn;
}
```

从上面的源码可以看出任务通知的机制非常简洁，与任务本身相关，将相关的信息填写到任务TCB之后，判断等待通知的任务优先级是否更高，如果更高则直接触发一次任务调度。由此，可以理解任务只能等待在一个任务通知上，也可以获得长度为1的消息内容，这也有别于Semaphore/Queue等方式，但是简洁高效是其最大的特点，在某些场景下可以使用此方式。

中断处理函数里的任务通知函数与非中断保护的函数类似，只是增加了开关中断保护；而等待任务通知不能在中断中执行，等待通知函数ulTaskNotifyTake()和xTaskNotifyWait()，直接看xTaskNotifyWait()

```c
BaseType_t xTaskNotifyWait(...)
{
  BaseType_t xReturn;

  taskENTER_CRITICAL();
  {
    //如果任务没有收到通知
    if( pxCurrentTCB->ucNotifyState != taskNOTIFICATION_RECEIVED )
    {
      pxCurrentTCB->ulNotifiedValue &= ~ulBitsToClearOnEntry;
      //将任务状态设置为等待通知状态
      pxCurrentTCB->ucNotifyState = taskWAITING_NOTIFICATION;
      if( xTicksToWait > ( TickType_t ) 0 )
      {
        //如果设置了阻塞等待超时时间，则将任务加入Delay List
        prvAddCurrentTaskToDelayedList( xTicksToWait, pdTRUE );
        //触发任务调度
        portYIELD_WITHIN_API();
      }
    }
  }
  taskEXIT_CRITICAL();
  //退出临界区，如果有任务调度，则任务被挂起

  taskENTER_CRITICAL();
  {
    if( pulNotificationValue != NULL )
    {
      //通知值赋值给函数传入参数，返回给调用者
      *pulNotificationValue = pxCurrentTCB->ulNotifiedValue;
    }

    if( pxCurrentTCB->ucNotifyState == taskWAITING_NOTIFICATION )
    {
      //没有收到通知值，可能是阻塞等待超时或者阻塞等待超时为0，直接返回错误
      xReturn = pdFALSE;
    }
    else
    {
      //收到了通知
      pxCurrentTCB->ulNotifiedValue &= ~ulBitsToClearOnExit;
      xReturn = pdTRUE;
    }

    //重置状态为非等待通知状态
    pxCurrentTCB->ucNotifyState = taskNOT_WAITING_NOTIFICATION;
  }
  taskEXIT_CRITICAL();
}
```

Task Notifications基本就阅读完成了，简洁高效而且占用RAM少，可以实现轻量级Queue，Binary Semaphore, Semaphore和Event Groups，具有很高的效率优势，但是也要注意使用限制，只能有1个任务接收通知，发送通知的任务不能因为无法发送通知而进入阻塞状态。
