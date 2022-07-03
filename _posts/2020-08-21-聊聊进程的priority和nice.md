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

![image](https://user-images.githubusercontent.com/36918717/177033216-412c06dd-9aa9-4dd6-834e-f29d1759b29c.png)
![image](https://user-images.githubusercontent.com/36918717/177033221-3c6d108d-593c-42a5-86b6-05ba71670c44.png)
![image](https://user-images.githubusercontent.com/36918717/177033225-a3dc4bd5-1c15-4637-ab6c-f4c8a766cbe3.png)

在sched_fork函数中会确定新进程的prio,static_prio,normal_prio等参数，并且对新进程的policy只会设置为非实时进程。从sched_fork代码可以看出新进程初始的nice值是0，所以初始static_prio的值=NICE_TO_PRIO(0)= nice+120=120

rt_priority: 实时优先级，由实时进程使用，普通进程没有用到该参数。取值范围为[0,99]。注意：rt_priority是值越大优先级越高。用户层可以通过系统调用函数sched_setscheduler()对其进行设置，该函数最终调用__setscheduler()。用户层可以通过系统调用函数sched_getparam()获取rt_priority的值。

normal_prio: normal_prio是基于前两个参数static_prio或rt_priority计算出来的。可以这样理解：static_prio和rt_priority分别代表普通进程和实时进程的”静态优先级”，代表进程的固有属性。由于他们两的表达方式不同，一个是值越小优先级越高，另一个是值越大优先级越高。有必要用normal_prio统一下。统一成值越小优先级越高。
![image](https://user-images.githubusercontent.com/36918717/177033229-8eb69064-e2a8-4970-b0f9-d2eeaf080ee9.png)
![image](https://user-images.githubusercontent.com/36918717/177033231-a0a40545-b36d-4c5f-9b75-2253f98f8b4e.png)
MAX_RT_PRIO=100

函数task_has_rt_policy判断进程p的调度策略是不是`SCHED_FIFO`和`SCHED_RR`中的一种，从而来判断它是不是实时进程，如果是实时进程，那么就返回`100-1-rt_priority`，如果是普通进程,不需要统一"单位",那么直接返回它的静态优先级`static_prio`。

所以针对实时进程，prio的值范围是[0,99]，且值越小优先级越高。和实时进程的rt_priority正好相反。

prio: 它表示进程的有效优先级(effectivepriority)，顾名思义，在内核中判断进程优先级时用的便是该参数，调度器考虑的优先级也就是它。其取值范围为[0,139]，值越小，优先级越低。并且nice系统调用改变的就是task_struct->prio值。
![image](https://user-images.githubusercontent.com/36918717/177033250-82fb0fbd-150e-449c-9291-3da1b879b6b9.png)
系统调用nice更改进程的nice值，不是直接将进程的nice加上increment，而是有一个复杂的计算逻辑：先取increment和-40的最大值，例如increment=-100，则max=-40；increment=10，则max=10。再取max和40的最小值，例如max=-40，则min=-40；max=10，则min=10，即increment介于[-40,40]之间。

上面4604行计算的nice就介于[static_prio-160,static_prio-80]之间。

上面4606行计算的nice介于[-20,19]之间。

然后调用set_user_nice修改进程的相关prio，首先修改static_prio=nice+120;其次修改进程有效优先级prio=effective_prio(p)，如果p是实时进程，则返回p->prio，否则返回p->normal_prio
![image](https://user-images.githubusercontent.com/36918717/177033262-8284db75-6873-464e-919a-980f65a480ed.png)
![image](https://user-images.githubusercontent.com/36918717/177033264-66923f68-6c66-4adc-a675-def9437c8506.png)

由于实时进程没有nice一说，因此nice只能针对非实时进程，即effective_prio函数返回p->normal_prio。非实时进程的normal_prio介于[100,139]之间。

 

举个例子：

如果一个 normal process 的  nice (NI) 是 0，那么他在 kernel 角度的 process priority

((nice) + DEFAULT_PRIO) = (0 + (MAX_RT_PRIO+ NICE_WIDTH / 2)) = (0 + 100 + 40 / 2) = 120

如果是 realtime process, 从 kernel 的角度看 p->prio (PR) 就等于 p->rt_priority。范围就是 [0, 99]。

 

总结：

进程控制块task_struct中的prio表示有效优先级，normal_prio表示动态优先级, static_prio表示静态优先级，rt_priority表示实时进程的优先级

 

static_prio:用于保存静态优先级, 是进程启动时分配的优先级, 并且可以通过nice和sched_setscheduler系统调用来进行修改, 否则在进程运行期间会一直保持恒定。

prio：进程的有效优先级, 进程调度也是根据prio进行选择调度的。

normal_prio：这个主要是对普通进程的static_prio和实时进程的rt_priority进行统一，对于非实时进程，static_prio范围是[100,139]，值越小优先级越高；对于实时进程rt_priority的范围是[0，99]，但值越大优先级越高，因此定义了normal_prio范围是[0，99]，值越小优先级越高。

rt_priority:实时进程的静态优先级，范围是[0,99]，且值越大优先级越高。





