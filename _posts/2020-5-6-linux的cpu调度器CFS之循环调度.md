---
layout:     post
title:      linux的cpu调度器CFS之循环调度

subtitle:   linux的cpu调度器CFS之循环调度

date:       2020-05-06
author:     lwk
catalog: true
tags:
    - linux
    - CFS
    - 循环调度
---

上一篇聊一聊linux的cpu调度器CFS，我们已经分析了进程创建之后是如何设置相应数据项以及是否进行抢占调度等逻辑。除了进程创建会涉及到CFS，周期性调度也会使用到CFS。

周期性调度是指Linux定时周期性地检查当前任务是否耗尽当前进程的时间片，并检查是否应该抢占当前进程。一般会在定时器的中断函数中，通过一层层函数调用最终到scheduler_tick()函数。

![image](https://user-images.githubusercontent.com/36918717/177022851-94ec8aad-42b0-4e45-8bd5-830764b13767.png)
由于调用task_tick的struct是sched_class，因此对应的回调函数是task_tick_fair
