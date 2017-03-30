---
layout: post
title: "RTOS学习笔记-FreeRTOS源码"
categories: RTOS
tags: ARM RTOS FreeRTOS Cortex-M
author: Will
---

* content
{:toc}


通过FreeRTOS官网资料，已经可以很好的使用FreeRTOS了，再深入的理解就需要深入到源码层，接下来就阅读源码，版本v9.0.0。

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
  //Task在Ready/Blocked/Suspended某个状态时，相应的List会指向这个ListItem_t
  vListInitialiseItem( &( pxNewTCB->xStateListItem ) );
  //Event相关，task相关的某个Event List会指向这个ListItem_t
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
  orrr14, #0xd  //0x0d:返回后进入线程模式
  bx r14
}
```

三个特殊的中断及中断处理函数，Systick/SVC/PendSV，这也是在移植FreeRTOS时需要特别注意的地方，Systick产生RTOS需要的Tick中断，其中断处理函数与RTOS密切相关，SVC如上所示启动Task，PendSV用于task调度切换，Systick/PendSV配置成了最低优先级，在中断抢占的前提下，PendSV被抢占但是在处理完高优先级的任务后，依然会进入中断处理函数处理，这样不会打断高优先级的任务，也能完成任务的调度切换。
