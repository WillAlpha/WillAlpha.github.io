---
layout: post
title: "ARM学习笔记-STM32"
categories: Embedded
tags: ARM MDK Cortex-M STM32
author: Will
---

* content
{:toc}


物联网以及智能硬件领域的微控制器，ARM Cortex-M系列无疑是首选，刚好最近从朋友处拿到一块STM32F4系列的开发板（NUCLEO STM32F412ZGT6），学习一下。

## STM32
STM32是意法半导体制作的基于自家设计的ARM Cortex-M内核处理器的评估开发板，提供了丰富的文档、开发工具以及开发示例，完善的社区支持，并且支持大量的扩展版，可以快速的构建产品原型，加快后续产品化。意法半导体还提供了stm32CubeMx软件，旨在最大化的减轻开发工作，通过此软件可以选择不同的处理器，解决引脚冲突，配置时钟树，设置外围外设，选择RTOS，选择中间件(USB/TCP/IP等)，生成面向IAR/KEIL/GCC的不同的软件工程。总之，最大化消除硬件平台差异化带来的改动，最大程度减轻构建软件工程的时间，让开发者更好的集中在自己产品的原型构建上。

## 软件开发环境
ARM的软件开发环境基本可以分为3种：

* ARM自家的集成开发环境ARM MDK，收购KEIL之后推出
* 瑞典的国宝级软件IAR
* GCC，开源的不二选择，可以linux命令行使用也可以结合Eclipse使用，桌面版集成开发环境请参考[openSTM32](http://www.openstm32.org/)/[TrueSTUDIO](https://atollic.com/truestudio/)

先来看一下ARM自家的MDK，目前国内用的最多，资料丰富；GCC在后面的ARM+Linux（或Android）会用到；IAR与openSTM32暂时先搁置一下，有时间再看，先看最典型的两种。

目前ARM MDK与IAR都不是免费的，国内使用大家按照默认规则就好了，Google或百度基本都能搞定，ARM MDK安装版本v5.23，操作系统需要Win7以上。

STM32开发板支持ST-Link/V2-1加载调试仿真，已经成为标配，只需要一根手机充电线（即USB线）连接开发板与PC，不需要再额外购买调试仿真器（其他常用的是JLINK和ULINK）。需要在PC上安装[ST-Link/V2-1 USB驱动v1.01](http://www.stmcu.com.cn/Designresource/design_resource_detail?file_name=STSW_LINK008&lang=EN&ver=1.01)。使用ST-Link/V2-1还需要在ARM MDK中配置使用ST-Link，稍后调试使用时再详细介绍。

## 官方标准固件库和软件包
STM32F4系列的官方标准固件库可以从STM32中文官网下载[v1.8.0](http://www.stmcu.com.cn/Designresource/design_resource_detail?file_name=STSW_STM32065&lang=EN&ver=1.8.0)。

解压缩标准固件开发包，可见目录结构：

```
_htmresc       
Libraries\  \\库文件，创建工程时用到
 |-CMSIS    \\ARM控制器软件接口标准文件
 |-STM32F4xx_StdPeriph_Driver  \\标准外设驱动，基于CMSIS
Projects\
 |-STM32F4xx_StdPeriph_Examples  \\标准固件例程
 |-STM32F4xx_StdPeriph_Templates  \\创建项目工程模板
Utilities\  \\例程中用到的一些工具资源
 |-Media
 |-ST
 |-STM32_EVAL
 |-Third_Party
MCD-ST***.pdf
Release_Notes.html
stm32fxx_dsp_stdperiph_lib_um.chm  //整个开发包说明文档
```

由于我们使用的工具是MDK，可以进入Projects\STM32F4xx_StdPeriph_Templates\

```
Projects\STM32F4xx_StdPeriph_Templates\
 |-EWARM\   \\IAR工程
 |-MDK-ARM\  \\MDK工程
 |-SW4STM32\  \\SW4STM32工程
 |-TrueSTUDIO\  \\TrueSTUDIO工程
 |-main.c
 |-main.h
 |-stm32f4xx_config.h
 |-stm32f4xx_it.c
 |-stm32f4xx_it.h
 |-system_stm32f4xx.c
 |-Release_Notes.html
 |-readme.txt  \\关键信息！！！
```

首先阅读`readme.txt`，里面都已经写得很详细了，先看`“How to use it ?”`，直接看MDK-ARM那段，是对如何使用MDK工程的说明，其他工具用到后再看：

```
+ MDK-ARM
 -Open the Template.uvprojx project
 -Rebuild all files: Project->Rebuild all target files
 -Load project image: Debug->Start/Stop Debug Session
 -Run program: Debug->Run (F5)
```

而Templates目录下的`*.h`和`*.c`文件即为标准固件包里的例程，即Projects\STM32F4xx_StdPeriph_Examples\下面关于各个模块的标准例程，将其copy到此Templates目录下，再使用相应的工程来编译加载调试运行。具体可以参看每个例程的`readme.txt`文件，都有如何使用此例程的详细说明。以Projects\STM32F4xx_StdPeriph_Examples\ADC\ADC_DMA\为例：

```
Projects\STM32F4xx_StdPeriph_Examples\ADC\ADC_DMA\
 |-main.c
 |-main.h
 |-stm32f4xx_config.h
 |-stm32f4xx_it.c
 |-stm32f4xx_it.h
 |-system_stm32f4xx.c
```

阅读`readme.txt`文件：

```
Example Description  \\关于此例程的总体说明
...
Directory contents  \\目录文件说明
 -system_stm32f4xx.c   STM32F4xx system clock configuration file
 -stm32f4xx_conf.h     Library Configuration file
 -stm32f4xx_it.c       Interrupt handlers
 -stm32f4xx_it.h       Interrupt handlers header file
 -main.c               Main program
 -main.h               Main program header file
How to use it ?  \\如何使用此例程，copy到模板目录中，还有额外的Utilities中的文件也要被用到
 In order to make the program work, you must do the following:
 -Copy all source files from this example folder to the template folder under Project\STM32F4xx_StdPeriph_Templates
 -Open your preferred toolchain
 -Select the project workspace related to the used device
  -If "STM32F40_41xxx" is selected as default project Add the following files in the project source list:
  -Utilities\STM32_EVAL\STM3240_41_G_EVAL\stm324xg_eval.c
  -Utilities\STM32_EVAL\STM3240_41_G_EVAL\stm324xg_eval_lcd.c
  -Utilities\STM32_EVAL\STM3240_41_G_EVAL\stm324xg_eval_fsmc_sram.c
  -Utilities\STM32_EVAL\STM3240_41_G_EVAL\stm324xg_eval_ioe.c
 -Rebuild all files and load your image into target memory
 -Run the example
```

`readme.txt`文件已经非常详细的说明了例程结合模板工程如何使用了，其他例程就都会用了，下面我们按照工程模板中的`readme.txt`说明，打开MDK工程并编译运行加载默认的例程。

双击MDK工程文件 Projects\STM32F4xx_StdPeriph_Templates\MDK-ARM\Project.uvprojx:
![打开工程图片](images/mdk_default_example_prj.jpg)

从左侧`Project`栏里的目录结构看，`CMSIS`和`MDK-ARM`目录下的system_stm32f4xx.c和startup_stm32f412xg.s都是标准固件库STM32F4xx_DSP_StdPeriph_Lib_V1.8.0提供的，位于标准Libraries\CMSIS\Device\ST\STM32F4xx\Source\Templates目录下，是符合ARM [CMSIS](http://baike.baidu.com/item/CMSIS?sefr=enterbtn)软件接口标准的，startup_stm32f412xg.s是ARM MDK环境下的启动文件，汇编语言编写，摘录一段：

```
Reset_Handler  PROC
    EXPORT  Reset_Handler  [WEAK]
  IMPORT  SystemInit
  IMPORT  __main

    LDR     R0, =SystemInit
    BLX     R0
    LDR     R0, =__main
    BX      R0
    ENDP
```

先执行`SystemInit`函数，`SystemInit`函数实现就在system_stm32f4xx.c里，初始化系统时钟/PLL/Flash接口等，然后再跳转到`main`函数。

`SystemInit`函数名的定义也是符合CMSIS软件接口标准的，另外在标准固件库里面Libraries\CMSIS\Device\ST\STM32F4xx\Include\还有2个头文件stm32f4xx.h和system_stm32f4xx.h，其中stm32f4xx.h是寄存器相关的定义，而system_stm32f4xx.h则是`SystemInit`函数的头文件。至此，符合CMSIS的两个`*.c`和两个`*.h`文件基本就清楚了，在任何一个工程里都少不了这几个文件，而且ARM提出了CMSIS标准，避免了各种基于ARM内核的CPU在软件定义时的混乱，最终目的还是减少嵌入式软件总是耗费精力在平台迁移带来的修改。

图中的`Project`目录栏里面的`Doc`目录很好理解，就是每个例程的`readme.txt`文件，`User`目录里面就是例程里面的源文件了，而`Project`目录栏里面的STM32F4xx_StdPeriph_Driver目录则是标准固件库里提供的STM32F4xx_DSP_StdPeriph_Lib_V1.8.0\Libraries\STM32F4xx_StdPeriph_Driver

我们以流水灯为示例做一个简单的测试，此开发板的3个led灯是由GPIO的PIN0/PIN7/PIN14来控制的，因此使用Template里面的默认GPIO工程，main.c里main函数添加如下代码：

```c
/* GPIOB Peripheral clock enable */
RCC_AHB1PeriphClockCmd(RCC_AHB1Periph_GPIOB, ENABLE);
/* Configure PIN0/7/14 (led) */
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_7 | GPIO_Pin_14;
GPIO_InitStructure.GPIO_Speed = GPIO_Low_Speed;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_OUT;
GPIO_InitStructure.GPIO_OType = GPIO_OType_PP;
GPIO_InitStructure.GPIO_PuPd = GPIO_PuPd_NOPULL;  
GPIO_Init(GPIOB, &GPIO_InitStructure);
GPIO_WriteBit(GPIOB, GPIO_Pin_0 | GPIO_Pin_7 | GPIO_Pin_14, Bit_RESET);


/* Infinite loop */
while (1)
{
  Delay(50);
  GPIO_WriteBit(GPIOB, GPIO_Pin_0 | GPIO_Pin_7 | GPIO_Pin_14, Bit_SET);
  Delay(50);
  GPIO_WriteBit(GPIOB, GPIO_Pin_0 | GPIO_Pin_7 | GPIO_Pin_14, Bit_RESET);
}
```

然后按照`readme.txt`文档里面的操作步骤，Rebuild all target files -> Start/Stop Debug Session -> Run 的过程，就可以看到3个不同颜色的led灯不停的闪烁了。需要注意在MDK的`Options for Target`里面`Debug`配置页配置调试器为`ST-Link Debugger`

至此，使用标准固件库的基本方法就比较清晰了，如果要学习stm32的某个模块，可以用这样的方式学习标准固件库里提供的Example了，这种方式非常贴近底层，对深入学习非常有帮助。

## CubeMx

CubeMx软件是针对ST系列的MCU提供的图形化配置软件，其目的在于让开发工程师最大可能的减少在MCU底层软件的配置修改上，将精力集中于自己的业务上，从ST官方提供的资料看来，这也是其主推的一种方式。如果使用ST系列的MCU，这确实是一种比较好的使用方式，如果在一家芯片公司，使用的是ARM核的自家MCU，这种方式就无法使用了。

下载安装[STM32CubeMx v4.19.0](http://www.stmcu.com.cn/Designresource/design_resource_detail/file/46359/lang/EN/token/bcc6af35fbfa52707727fcd3ea7c203f)，还是以上面的标准固件库点亮3个led灯为示例，这次使用CubeMx软件生成MDK工程。

STM32CubeMx可参考官方使用说明[文档](http://www.stmcu.com.cn/Designresource/design_resource_detail/file/47493/lang/EN/token/36fac307b9a8f5fb43c1b1b0fbb0f997)，需要首先安装Java 1.7版本及以上，然后安装STM32CubeMx，安装完成后就可以打开使用了。首先安装相应的固件包，软件界面`Help` -> `Install New Libraries`，进入到下面的界面，选择STM32CubeF4 Releases，在这栏里面选择最新的版本安装，此固件包对应开发板STM32F4xx系列，后面通过STM32CubeMx定制工程软件时就会用到此固件包。
![安装stm32cubemxf4固件包](images/stm32cubemx_libraries_f4.jpg)

安装完成后，在STM32CubeMx主界面，点击`New Project`新建工程，看到下图所示，选择自己的开发板：
![stm32cubemx新建工程选择开发板](images/stm32cubemx_select_board.jpg)

选好开发板后，就可以看到图形配置页面了：
![stm32cubemx工程配置页面](images/stm32cubemx_project_config.jpg)

由于看原理图知道控制LED的3个GPIO管脚是PB0/PB7/PB14，因此可以直接在`Pinout`页面的芯片的图片的这3个管脚上直接选择成`GPIO_Output`，如下图所示：
![stm32cubemx的控制LED的GPIO配置](images/stm32cubemx_gpio_config.jpg)

依次`Clock Configuration` `Configuration` `Power Consumption Calculator` 暂时维持原配置不变，点击`Generate source code based on user settings`按钮，生成ARM MDK V5的工程：
![stm32cubemx生成code配置](images/stm32cubemx_gen_code_config.jpg)

打开生成的工程，如下图：
![打开stm32cubemx生成的led闪烁工程](images/stm32cubemx_gen_mdk_prj.jpg)

左侧工程栏与上面标准固件包工程非常相似，只是有些替换成了HAL层的文件名，HAL层是ST抽象出来的一层软件，为了配合CubeMx软件灵活的配置生成工程等；main.c中的main函数可以看到初始化函数都写好了，而且注释可以看出添加用户代码的地方，在while(1)循环里添加循环点亮LED的代码：

```c
/* Infinite loop */
/* USER CODE BEGIN WHILE */
while (1)
{
/* USER CODE END WHILE */

/* USER CODE BEGIN 3 */
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0|GPIO_PIN_14|GPIO_PIN_7, GPIO_PIN_SET);
  HAL_Delay(100);
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0|GPIO_PIN_14|GPIO_PIN_7, GPIO_PIN_RESET);
  HAL_Delay(100);
}
/* USER CODE END 3 */
```

与前面标准固件库示例类似，通过ARM MDK 5编译软件并下载到开发板，就可以看到3个LED等不停的闪烁。
