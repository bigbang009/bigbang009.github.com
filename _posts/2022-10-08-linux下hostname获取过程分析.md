---
layout:     post
title:      linux下hostname获取过程分析
subtitle:   linux下hostname获取过程分析
date:       2022-10-08
author:     lwk
catalog: true
tags:
    - linux
    - kernel
    - hostname
---

hostname是centos内置的获取主机名的命令，该命令对应的源码是net-tools
[root@] /data0/src/os/net-tools$ ls hostname*
hostname  hostname.c  hostname.o

其中hostname二进制是编译出来的可执行文件
通过GDB hostname可以看到，最终是通过系统调用gethostname获取主机名
![image](https://user-images.githubusercontent.com/36918717/194737638-fc8b0eb7-72b0-4ccb-9c51-b08650ed6605.png)
看下内核源码，本文以内核5.5.0版本为主
![image](https://user-images.githubusercontent.com/36918717/194737643-ffe5ff5f-b961-4a04-9260-86f6664bedfa.png)
gethostname通过utsname获取对应nodename，结构是struct new_utsname
![image](https://user-images.githubusercontent.com/36918717/194737645-ab78c51a-377f-4db1-81ba-4a693b65767b.png)
从utsname函数可以看出，是通过获取current这个task_struct的nsproxy->uts_ns->name成员从而返回主机信息
![image](https://user-images.githubusercontent.com/36918717/194737652-a679dc67-bccf-426a-ac48-6f4737483d4f.png)
![image](https://user-images.githubusercontent.com/36918717/194737655-db8f84c0-2293-42a2-922f-de8a5afe13ec.png)
其中count表示的是进程数，增加一个进程count数就加一
![image](https://user-images.githubusercontent.com/36918717/194737660-2aea9d1a-e982-4d2d-b011-dab2bde03954.png)
![image](https://user-images.githubusercontent.com/36918717/194737665-a444bed8-eba2-43ff-8611-a98bff9534b8.png)

这里还涉及到一个kernel的知识点就是namespace，hostname涉及到了kernel的UTS namespace，关于namespace本文不做阐述，以后有机会再详细分析
