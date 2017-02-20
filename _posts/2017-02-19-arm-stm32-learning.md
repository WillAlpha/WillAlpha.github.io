---
layout: post
title: "ARM学习笔记-STM32"
categories: Embedded
tags: ARM MDK Cortex-M STM32
author: Will
---

* content
{:toc}


物联网以及智能硬件领域的微控制器，ARM Cortex-M系列无疑是首选，刚好最近从朋友处拿到一块STM32F4系列的开发板（Cortex-M4），学习一下。

## STM32
STM32是意法半导体制作的基于自家设计的ARM Cortex-M内核处理器的评估开发板，提供了丰富的文档、开发工具以及开发示例，可以快速的构建产品原型，以及后续产品化。意法半导体还提供了CubeMx软件，旨在最大化的减轻开发工作，通过此软件可以选择不同的处理器，解决引脚冲突，配置时钟树，设置外围外设，选择RTOS，选择中间件(USB/TCP/IP等)，生成面向IAR/KEIL/GCC的不同的软件工程。总之，最大化消除硬件平台差异化带来的改动，最大程度减轻构建软件工程的时间，让开发者更好的集中在自己产品的原型构建上。

## 构建软件开发环境
ARM的软件开发环境基本可以分为3种：

* ARM自家的集成开发环境ARM MDK，收购KEIL之后推出，国内用的最多
* 瑞典的国宝级软件IAR
* GCC，开源的不二选择，可以linux命令行使用也可以结合Eclipse使用

先来看一下ARM自家的MDK，目前国内用的最多，资料丰富；GCC在后面的ARM+Linux（或Android）会用到；IAR暂时先搁置一下，有时间再看，先看最典型的的两种。

目前ARM MDK与IAR都不是免费的，国内使用大家按照默认规则就好了，Google或百度基本都能搞定，ARM MDK安装版本v5.23，操作系统需要Win7以上。

