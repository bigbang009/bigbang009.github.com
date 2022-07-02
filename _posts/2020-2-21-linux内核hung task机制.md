---
layout:     post
title:      linux内核hung task机制
subtitle:   linux内核hung task机制
date:       2020-02-21
author:     lwk
catalog: true
tags:
    - linux
    - hungtask
---

今天说下内核的hung task机制。

有时在/var/log/messages会看到这样的提示：

kernel: INFO: task xxx blocked for morethan 120 seconds

kernel: "echo 0 >/proc/sys/kernel/hung_task_timeout_secs" disables this message.

kernel: xxx D ffff944c19515140     0 22008      1 0x00000080

 

hung task源码在kernel/hung_task.c文件中，系统启动时会执行hung_task_init

![image](https://user-images.githubusercontent.com/36918717/177003540-38bd8b00-6bbe-4aaf-af59-aa360a20f45c.png)

Kthread_run创建一个名为khungtaskd的内核线程，处理函数是watchdog()


![image](https://user-images.githubusercontent.com/36918717/177003547-51309e49-20cb-4429-856c-c6c218f24c90.png)

其中，和hung task相关的内核参数可以通过sysctl –a | grep hung查看

kernel.hung_task_check_count = 4194304

kernel.hung_task_panic = 1

kernel.hung_task_timeout_secs = 120

kernel.hung_task_warnings = 10

 

watchdog函数调用check_hung_uninterruptible_tasks，遍历所有进程和线程，对处于D状态的进程(线程)进行处理。

![image](https://user-images.githubusercontent.com/36918717/177003552-972d2208-0d30-4e30-bf5a-01b60c4dc6a4.png)

![image](https://user-images.githubusercontent.com/36918717/177003559-1705d6b6-a632-4037-a637-c82e35bf6c2f.png)

![image](https://user-images.githubusercontent.com/36918717/177003561-dd5b7e6c-20d6-4b85-8dfa-59134790c36a.png)

内核有进程处于D状态在120s内都没有被调度，则默认会触发panic，如果要关闭hung task panic，则可以设置内核参数kernel.hung_task_panic=0进行关闭。






