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

由于我们使用的工具是MDK，可以进入`Projects\STM32F4xx_StdPeriph_Templates\`

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

而Templates目录下的`*.h`和`*.c`文件即为标准固件包里的例程，即`Projects\STM32F4xx_StdPeriph_Examples\`下面关于各个模块的标准例程，将其copy到此Templates目录下，再使用相应的工程来编译加载调试运行。具体可以参看每个例程的`readme.txt`文件，都有如何使用此例程的详细说明。以`Projects\STM32F4xx_StdPeriph_Examples\ADC\ADC_DMA\`为例：

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

双击MDK工程文件 `Projects\STM32F4xx_StdPeriph_Templates\MDK-ARM\Project.uvprojx`:
![打开工程图片](images/mdk_default_example_prj.jpg)
