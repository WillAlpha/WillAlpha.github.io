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
STM32是意法半导体制作的基于自家设计的ARM Cortex-M内核处理器的评估开发板，提供了丰富的文档、开发工具以及开发示例，完善的社区支持，并且支持大量的扩展版，可以快速的构建产品原型，加快后续产品化。意法半导体还提供了CubeMx软件，旨在最大化的减轻开发工作，通过此软件可以选择不同的处理器，解决引脚冲突，配置时钟树，设置外围外设，选择RTOS，选择中间件(USB/TCP/IP等)，生成面向IAR/KEIL/GCC的不同的软件工程。总之，最大化消除硬件平台差异化带来的改动，最大程度减轻构建软件工程的时间，让开发者更好的集中在自己产品的原型构建上。

## 软件开发环境
ARM的软件开发环境基本可以分为3种：

* ARM自家的集成开发环境ARM MDK，收购KEIL之后推出，国内用的最多
* 瑞典的国宝级软件IAR
* GCC，开源的不二选择，可以linux命令行使用也可以结合Eclipse使用，桌面版集成开发环境请参考[openSTM32](http://www.openstm32.org/)/[TrueSTUDIO](https://atollic.com/truestudio/)

先来看一下ARM自家的MDK，目前国内用的最多，资料丰富；GCC在后面的ARM+Linux（或Android）会用到；IAR与openSTM32暂时先搁置一下，有时间再看，先看最典型的两种。

目前ARM MDK与IAR都不是免费的，国内使用大家按照默认规则就好了，Google或百度基本都能搞定，ARM MDK安装版本v5.23，操作系统需要Win7以上。

STM32开发板支持ST-Link/V2-1加载调试仿真，已经成为标配，只需要一根手机充电线（即USB线）连接开发板与PC，不需要再额外购买调试仿真器（其他常用的是JLINK和ULINK）。需要在PC上安装[ST-Link/V2-1 USB驱动v1.01](http://www.stmcu.com.cn/Designresource/design_resource_detail?file_name=STSW_LINK008&lang=EN&ver=1.01)。使用ST-Link/V2-1还需要在ARM MDK中配置使用ST-Link，稍后调试使用时再详细介绍。

## 官方标准固件库和软件包
STM32F4系列的官方标准固件库可以从STM32中文官网下载[v1.8.0](http://www.stmcu.com.cn/Designresource/design_resource_detail?file_name=STSW_STM32065&lang=EN&ver=1.8.0)。

