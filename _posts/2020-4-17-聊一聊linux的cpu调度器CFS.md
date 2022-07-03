---
layout:     post
title:      聊一聊linux的cpu调度器CFS
subtitle:   聊一聊linux的cpu调度器CFS
date:       2020-04-17
author:     lwk
catalog: true
tags:
    - linux
    - cfs
    - 进程调度
---

之前文章linux内核fork剖析分析了fork模式创建进程。其中在copy_process函数中sched_fork主要实现父子进程的时间片分割。本文主要分析linux调度器之CFS的逻辑，包括调度算法和数据结构等。有益于linux内核学习者对linux的CPU调度有一定的认识和熟悉。

![image](https://user-images.githubusercontent.com/36918717/177022643-7df0357c-6e10-4343-a574-f13740bdeacd.png)
![image](https://user-images.githubusercontent.com/36918717/177022646-5ed798b0-3490-4093-9a51-199dda0c7dee.png)
![image](https://user-images.githubusercontent.com/36918717/177022648-7cc056de-88a2-4ca9-8062-47aaecf55cbd.png)

task_fork是一个函数指针，实际调用的是fair_sched_class的task_fork_fair函数，位于文件kernel/sched/fair.c中。

![image](https://user-images.githubusercontent.com/36918717/177022650-56a558c4-f964-4c4d-a55c-ab68d97a05fe.png)
每个cpu的运行队列
![image](https://user-images.githubusercontent.com/36918717/177022652-2b8d1abb-d5f7-4710-b221-5aaa6cf15c38.png)
CFS运行队列，cfs_rq->tasks_timeline是个RB-Tree

![image](https://user-images.githubusercontent.com/36918717/177022655-2ed85cc6-c797-43fd-8592-00c621434f4a.png)
CFS调度实体
![image](https://user-images.githubusercontent.com/36918717/177022659-677118e7-763c-4d76-ae37-fd06d31d88ba.png)
CFS调度器涉及到的RB-Tree结构在include/linux/rbtree.h
![image](https://user-images.githubusercontent.com/36918717/177022660-e82dd2d0-6b94-4a13-a5dc-b84dc4e680ea.png)
![image](https://user-images.githubusercontent.com/36918717/177022663-e6b2752f-cfda-4285-aceb-be299c9a374a.png)
task_fork_fair函数
![image](https://user-images.githubusercontent.com/36918717/177022666-9c0e0a05-9dcd-4f11-9c83-88a6f703868a.png)
update_curr函数主要是更新队列调度时间





![image](https://user-images.githubusercontent.com/36918717/177022671-d29aafa1-a38d-49a3-8fd0-313ad6b87737.png)


update_min_vruntime()函数保证就绪队列的最小虚拟时间min_vruntime单调递增的特性，更新最小虚拟时间。

![image](https://user-images.githubusercontent.com/36918717/177022675-0e1ce406-b947-4f20-b367-53a4da149503.png)

继续分析place_entity函数

![image](https://user-images.githubusercontent.com/36918717/177022679-70ec7474-befe-40fa-a805-aaa2f700bf98.png)
惩罚值计算逻辑：
![image](https://user-images.githubusercontent.com/36918717/177022681-fcbd9123-28b9-442f-8fb7-cafe863f1e16.png)
![image](https://user-images.githubusercontent.com/36918717/177022683-a6c92b7b-1cb3-4184-aaea-1403d59bf42a.png)

do_fork的初始化工作就完成了。接下来就是将进程加入就绪队列，等待唤醒调度。

![image](https://user-images.githubusercontent.com/36918717/177022685-4d7fdb49-3bac-4d63-9369-e00e0948707a.png)
wake_up_new_task函数
![image](https://user-images.githubusercontent.com/36918717/177022687-fc603d65-023b-4375-af37-0284faa6ac99.png)
![image](https://user-images.githubusercontent.com/36918717/177022689-dd80c3d0-2f4d-4866-8811-389b5b5f7d9d.png)
![image](https://user-images.githubusercontent.com/36918717/177022692-a35d4d98-f45f-49f3-ad46-9762f60ddf48.png)

enqueuer_task_fair函数

![image](https://user-images.githubusercontent.com/36918717/177022698-e4af746d-dd85-4e76-becd-6a23831b29a6.png)
接下来重点分析入队函数enqueue_entity：
![image](https://user-images.githubusercontent.com/36918717/177022705-f5c0fcc2-2ba5-4f05-a714-57d3464e18f5.png)
accout_entity_enqueue函数更新就绪队列相关信息

![image](https://user-images.githubusercontent.com/36918717/177022708-abf357c4-b45e-47b8-a544-840d513f8129.png)

接下来就是check_preempt_curr函数：检查是否满足抢占条件
![image](https://user-images.githubusercontent.com/36918717/177022710-71f83671-e2b3-49bd-bb6c-e37099e83558.png)
情况1：唤醒的进程和当前running的进程属于同一个调度类，例如CFS

Check_preempt_curr实际是回调函数，对应函数是check_preemt_wakeup

![image](https://user-images.githubusercontent.com/36918717/177022715-3af13b4a-b5c4-4d94-bebd-9d4e17caa450.png)
![image](https://user-images.githubusercontent.com/36918717/177022717-022e2807-d607-42cc-94a8-937aacccf7bc.png)
![image](https://user-images.githubusercontent.com/36918717/177022720-5bfca69f-b580-4cdd-b983-daa6b76b792a.png)
![image](https://user-images.githubusercontent.com/36918717/177022723-d593fc49-ef0a-4ddf-b2ce-b344470d0bbd.png)
情况2：唤醒的进程和当前running进程不属于同一个调度类，例如唤醒进程属于RT，当前running进程属于CFS
![image](https://user-images.githubusercontent.com/36918717/177022726-e446f55d-eac1-42b0-923c-3f84106f9d91.png)

总结：本文主要介绍CFS实现细节，对于学习linux 内核有比较好的帮助，尤其在定位系统级问题时大有裨益。另外，学习linux内核也能获取不少系统设计方面的知识，包括算法和一些数据结构，为以后系统设计和编码上面提供参考。

