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
![image](https://user-images.githubusercontent.com/36918717/177023610-54c3246d-0b42-4f62-b890-2d735ff7daf9.png)
![image](https://user-images.githubusercontent.com/36918717/177023612-5a7fd72c-f5ba-47b4-a34b-c1d97c61786d.png)
![image](https://user-images.githubusercontent.com/36918717/177023615-673c6146-9d45-4993-9a20-68668722ad4a.png)
update_curr函数已经在上一篇文章中分析过来，因此本文重点分析check_preemt_tick函数。

![image](https://user-images.githubusercontent.com/36918717/177023619-6708aa82-f461-45d2-bbe9-73ff75b733c0.png)
针对周期性调度可以总结为：

1、更新正在运行进程的虚拟时间

2、检查当前进程是否满足被抢占，满足则将从就绪队列挑选运行时间最小的进程；将被强占的进程重新加入到RB-tree中；从RB-tree中删除即将运行的调度实体。

 

如何选择下一个合适进程运行

![image](https://user-images.githubusercontent.com/36918717/177023623-49db7828-9c34-478a-a563-a633f408cb2b.png)
__schedule核心逻辑：


![image](https://user-images.githubusercontent.com/36918717/177023628-2eb32be8-bdbb-4822-97bb-d464b37c8856.png)

![image](https://user-images.githubusercontent.com/36918717/177023630-72711d1a-8503-4fdd-863c-f6fd021337e1.png)
![image](https://user-images.githubusercontent.com/36918717/177023632-3673ed6a-f4b7-4165-93cc-3cf5a6c6a645.png)
![image](https://user-images.githubusercontent.com/36918717/177023635-6bba2719-fd25-432d-aede-48da9a7ff55b.png)

![image](https://user-images.githubusercontent.com/36918717/177023637-2e5cd90d-37e8-40d1-ac52-4cfdc9f9eccc.png)
put_prev_task最终回调put_prev_task_fair.

![image](https://user-images.githubusercontent.com/36918717/177023645-08b7470b-e599-40a8-9d10-e6968839e6cd.png)

看下put_prev_task_fair的实现逻辑。

![image](https://user-images.githubusercontent.com/36918717/177023648-235b5b43-2368-4e17-9a21-c8c1ec1a0cbe.png)

put_prev_task_fair函数里主要逻辑是put_prev_entity函数，看下put_prev_entity函数的实现逻辑。
![image](https://user-images.githubusercontent.com/36918717/177023650-cd0e4726-d658-4bde-a2c5-8355053baed5.png)

prev进程的已经处理完毕，接下来就是调用set_next_entity函数更新即将运行的调度实体

![image](https://user-images.githubusercontent.com/36918717/177023653-68f86d0e-8fa5-44c0-b26d-0a7dc3ee771e.png)
在__schedule()函数中，如果prev进程主动睡眠。那么会调用deactivate_task()函数。deactivate_task()函数最终会调用调度类dequeue_task方法。CFS调度类对应的函数是dequeue_task_fair()，该函数是enqueue_task_fair()函数反操作。
![image](https://user-images.githubusercontent.com/36918717/177023657-7ac95dce-63ef-4bc0-9a06-7556c8924895.png)

![image](https://user-images.githubusercontent.com/36918717/177023662-9a7e51af-206c-4cf9-bc19-45bafc3d5410.png)

![image](https://user-images.githubusercontent.com/36918717/177023663-7a16bf85-23b7-475b-b0b7-b0f2fe36f274.png)
![image](https://user-images.githubusercontent.com/36918717/177023664-fe509261-f3c1-46ea-82f1-3a7e86afe807.png)
dequeuer_task_fair函数
![image](https://user-images.githubusercontent.com/36918717/177023667-de65609f-ef76-438f-8706-52fbfb5ac8e7.png)


![image](https://user-images.githubusercontent.com/36918717/177023669-43c70adb-7e48-49c9-8b3d-01601672be41.png)

![image](https://user-images.githubusercontent.com/36918717/177023672-c2430596-cde6-4cf5-934e-b85fda9b7213.png)
总结：本文主要是继上一篇CFS调度的续篇，主要阐述了CFS的周期性调度逻辑。对于了解CPU调度逻辑（包括进程什么时候被调度，什么时候被强占，调度时的状态，未调度时的状态，数据存储、计算和更新等知识有一个整体的概念和把握）。另外，理解CPU调度逻辑对于系统问题排查也很有帮助。
