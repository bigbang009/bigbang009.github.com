---
layout:     post
title:      聊聊进程的priority和nice
subtitle:   聊聊进程的priority和nice
date:       2020-08-21
author:     lwk
catalog: true
tags:
    - linux
    - priority
	  - nice
---

在学习和实际工作中，经常会遇到进程的priority和nice，之前就一知半解，所以抽空又重新学习整理了下进程的priority和nice到底是什么意思，以及两者之间的关系。

首先要说起实时进程和非实时进程，系统（即内核）区分进程是不是实时进程是通过task_struct的policy进行区分，policy的取值如下：
![image](https://user-images.githubusercontent.com/36918717/177028177-54642ef7-7121-4407-915b-845d4f680fde.png)
其中，SCHED_NORMAL和SCHED_BATCH表示非实时进程，用于CFS调度；SCHED_FIFO和SCHED_RR表示实时进程。

实时进程是没有nice的，所以不可以设置实时进程的nice。而且nice的取值范围为[-20,19]，值越小，优先级越高。当fork一个进程时，进程的初始化priority和nice值默认是多少呢？

由之前对fork的分析，可知，在copy_process()函数中会调用sched_fork()，sched_fork函数对新进程设置了priority初始值，那nice值呢？其实在task_struct结构里其实没有nice结构项的，nice是通过priority计算得到的。

PRIO_TO_NICE((p)->static_prio
![image](https://user-images.githubusercontent.com/36918717/177028185-3e0ad7ae-0c7a-4c37-b387-49ff3fdd4ca0.png)


即nice通过PRIO_TO_NICE宏进行计算，例如当static_prio=120，则nice=120-120=0
![image](https://user-images.githubusercontent.com/36918717/177028188-bde177bb-5585-472e-b44a-5ce7af722181.png)
在内核的task_struct结构定义了4个优先级：

static_prio: 静态优先级，由普通进程(非实时进程)使用，静态优先级是进程启动时分配的优先级（sched_fork函数中确定的）。可以用nice()和sched_setscheduler()系统调用修改，否则在进程运行过程中一直保持恒定。注意，实时进程没有用到该参数。

该字段的取值范围为[100,139]([MAX_RT_PRIO,MAX_PRIO-1])，值越小优先级越高。

 

但我们知道当我们调用nice()函数的时候，传入的nice的值的范围是[-20,19]，所以在nice()函数内部该系统调用函数对应内核级别的系统调用sys_nice()，并进而调用set_user_nice()函数，该函数内部会将nice()函数传入的[-20,19]范围的值映射为[100,139]之间的值。nice值和static_prio之间存在以下映射关系：

static_prio = nice + 20 + MAX_RT_PRIO，MAX_RT_PRIO=100，因此static_prio范围是[100,139]

 

静态优先级static_prio(普通进程)和实时优先级rt_priority(实时进程)是计算的起点，因此他们也是进程创建的时候设定好的。
