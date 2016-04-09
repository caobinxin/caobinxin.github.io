---
layout: page
title: 
permalink: /android/
---

----------


> 基于Android 6.0源码，深入剖析Android系统架构，争取各个击破。这样才能犹如庖丁解牛，解决、分析问题则能游刃有余。

## 一、简述

Android系统非常庞大、错中复杂，其底层是采用Linux作为基底，上层采用包含虚拟机的Java层以及Native层，通过系统调用(Syscall)连通系统的内核空间与用户空间。用户空间主要采用C++和Java代码，通过JNI技术打通用户空间的Java层和Native层(C++/C)，从而融为一体。

Google官方提供了一张[经典的四层架构图](http://gityuan.com/2016/01/30/android-boot/#android)，从下往上依次分为Linux内核、系统库和Android运行时环境、框架层以及应用层这4层架构，其中每一层都包含大量的子模块或子系统。这只是如垒砖般地简单分层，还远不足以表达Android整个系统的内部架构，运行机理，以及各个模块之间是如何衔接与配合工作的。为了更深入地掌握Android整个架构思想以及各个模块在Android系统所处的地位与价值，计划以Android系统启动过程为主线，**以进程的视角**来诠释Android系统全貌，全方位的深度剖析各个模块功能，争取各个击破。

## 二、Android架构

**系统启动架构图**

点击查看[大图](http://gityuan.com/images/android-process/android-boot.jpg)

![process_status](/images/android-process/android-boot.jpg)
  
  
**图解：**  
Android系统启动过程由上图从下往上的一个过程：`Loader` -> `Kernel` -> `Native` -> `Framework` -> `App`，接来下简要说说每个过程：

### 2.1 Loader层

- Boot ROM: 当手机处于关机状态时，长按Power键开机，引导芯片开始从固化在`ROM`里的预设出代码开始执行，然后加载引导程序到`RAM`；
- Boot Loader：这是启动Android系统之前的引导程序，主要是检查RAM，初始化硬件参数等功能。
	
### 2.2 Kernel层

到这里才刚刚开始进入Android系统.

- 启动Kernel的0号进程：初始化进程管理、内存管理，加载Display,Camera Driver，Binder Driver等相关工作；
- 启动kthreadd进程（pid=2）：是Linux系统的内核进程，会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程。`kthreadd进程是所有内核进程的鼻祖`。
	
### 2.3 Native层

启动init进程(pid=1),是Linux系统的用户进程，`init进程是所有用户进程的鼻祖`。

- init进程启动`Media Server`(多媒体服务)、`servicemanager`(binder服务管家)、`bootanim`(开机动画)等重要服务
- init进程还会孵化出installd(用于App安装)、ueventd、adbd、lmkd(用于内存管理)等用户守护进程；
- init进程孵化出Zygote进程，Zygote进程是Android系统的第一个Java进程，`Zygote是所有Java进程的父进程`，Zygote进程本身是由init进程孵化而来的。

### 2.4 Framework层

- Zygote进程，是由init进程通过解析init.rc文件后fork生成的，Zygote进程主要包含：
	- 加载ZygoteInit类，注册Zygote Socket服务端套接字；
	- 加载虚拟机；
	- preloadClasses；
	- preloadResouces。
- Zygote进程fork出System Server进程，`System Server是Zygote孵化的第一个进程`，地位非常重要。
- System Server进程：负责启动和管理整个Java framework，包含ActivityManager，PowerManager等服务。
- Media Server进程：负责启动和管理整个C++ framework，包含AudioFlinger，Camera Service等服务。
	
### 2.5 App层

- Zygote进程孵化出的第一个App进程是Launcher，这是用户看到的桌面App；
- Zygote进程还会创建Browser，Phone，Email等App进程，每个App至少运行在一个进程上。
- 所有的App进程都是由Zygote进程fork生成的。

##  三、IPC机制

无论是Android系统，还是各种Linux衍生系统，各个组件、模块往往运行在各种不同的进程和线程内，这里就必然涉及进程/线程之间的通信(IPC)。Linux现有管道、消息队列、共享内存、套接字、信号量、信号这些IPC机制，Android额外还有Binder IPC机制，Android OS中的Zygote进程的IPC采用的是Socket（套接字）机制，Android中的Kill Process采用的signal（信号）机制等等，在上层system server、media server以及上层App之间更多的是采用Binder IPC方式来完成跨进程间的通信。


### 3.1 Binder 

Binder作为Android系统提供的一种IPC机制，无论从系统开发还是应用开发，都是Android系统中最重要的组成，也是最难理解的一块知识点，想了解[为什么Android要采用Binder作为IPC机制？](https://www.zhihu.com/question/39440766/answer/89210950)，可查看博主在知乎上的回答。深入了解Binder机制，最好的方法便是阅读源码，借用Linux鼻祖Linus Torvalds曾说过的一句话：Read The Fucking Source Code。下面简要说说Binder IPC原理。

**Binder IPC原理**

Binder通信采用c/s架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。

![ServiceManager](/images/binder/prepare/IPC-Binder.jpg)
	
- 想进一步了解Binder，可查看[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)，Binder系列用了11篇文章，从驱动、native、framework、app四个层面，从源码角度出发来讲述整个过程。


##  四、计划提纲

2016年新的一年刚开始，首先祝大家、也祝自己在新的一年诸事顺心，事业蒸蒸日上。在过去的一年，对于Android从底层一路到上层有不少自己的理解和沉淀，但总体较零散，未成体系。借着今天（元旦假日的最后一天），给自己的新的一年提前做一个计划，把知识进行归档整理与再学习，从而加深对Android架构的理解。通过前面对系统启动的介绍，相信大家对Android系统有了一个整体观，接下来需要抓核心、理思路。


**（1）**Android系统启动过程中，有几个非常重要的进程：`init`、`Zygote`、`system_server`进程:

- [Android系统启动—init篇](http://gityuan.com/2016/02/05/android-init/)
- [Android系统启动—Zygote篇](http://gityuan.com/2016/02/13/android-zygote/)
- Android系统启动—SystemServer篇
	- [SystemServer上篇](http://gityuan.com/2016/02/14/android-system-server/)
	- [SystemServer下篇](http://gityuan.com/2016/02/20/android-system-server-2/)
  
**（2）**再则就是在整个架构中有大量的服务，都是基于Binder来交互的，为了搞清楚binder，用了13篇文章来讲解[Binder](http://gityuan.com/2015/10/31/binder-prepare/)，从binder驱动到应用层整个完整的流程。针对比较核心服务来重点分析，计划分别用文章来对核心服务展开剖析：

- Android服务篇-ActivityManagerService
	- [AMS启动过程（一）](http://gityuan.com/2016/02/21/activity-manager-service/)
- Android服务篇-PackageManagerService
- Android服务篇-PowerManagerService
- Android服务篇-BatteryService
	- [Android耗电统计算法](http://gityuan.com/2016/01/10/power_rank/)
- Android服务篇-WindowManagerService
  
当然graphic也是一大块难啃的模块，也是需要整理的，先留个空位吧。

**（3）**对于App来说，Android应用的四大组件Activity，Service，Broadcast Receiver， Content Provider最为核心，那么我们需要分别展开对其他的分解：

- Android组件-Activity
	- [startActivity流程分析(一)](http://gityuan.com/2016/03/12/start-activity/)
- Android组件-Service
	- [startService流程分析](http://gityuan.com/2016/03/06/start-service/)
- Android组件-Broadcast Receiver
- Android组件-Content Provider

  
**（4）**有了这些，中间还缺少关于虚拟机ART的介绍，会需要对ART分析，后续还需要开展对ART虚拟机的一系列文章。回顾整个架构，谈谈系统性能，需要先掌握进程、内存、IO这些层面知识，这里牵涉面较广，从底层Linux层直至上层App

- 进程篇
	- [理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/)
	- [Android进程整理](http://gityuan.com/2015/12/19/android-process-category/)
	- [Android进程生命周期与ADJ](http://gityuan.com/2015/10/01/process-lifecycle/)
	- [进程优先级](http://gityuan.com/2015/10/01/process-priority/)
- 内存篇
	- [Linux内存管理](http://gityuan.com/2015/10/30/kernel-memory/)
- IO篇
- Linux驱动篇
 
**（5）**最后，说说Android相关的一些常用命令和工具

- [理解Android编译命令](http://gityuan.com/2016/03/19/android-build/)
- [性能工具Systrace](http://gityuan.com/2016/01/17/systrace/)
- [Android内存分析命令](http://gityuan.com/2016/01/02/memory-analysis-command/)
- [ps进程命令](http://gityuan.com/2015/10/11/ps-command/)
- [Am命令用法](http://gityuan.com/2016/02/27/am-command/)
- [Pm命令用法](http://gityuan.com/2016/02/28/pm-command/)
  
  
本博客还有很多文章并没有写到上面这个清单，先写这么多，后续再不断更新与完善。最后，欢迎大家交流与纠错，大家来找茬。

----------

欢迎关注我的**[微博：Gityuan](http://weibo.com/gityuan)**，微信公众号：gityuan，后面会持续分享更多原创技术干货。