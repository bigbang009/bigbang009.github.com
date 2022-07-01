---
layout:     post
title:      说一说linux的crontab执行逻辑
subtitle:   说一说linux的crontab执行逻辑
date:       2020-01-06
author:     lwk
catalog: true
tags:
    - linux
    - crontab
---


很多时候都要使用到crontab定时执行任务，但你知道crontab到底是怎么执行的吗？接下来就给大家讲解一下crontab执行逻辑。

首先我们把源码clone下来。https://github.com/cronie-crond/cronie是crond的源码

 

首先我们找台机器，ps看下当前系统的cron进程信息
![image](https://user-images.githubusercontent.com/36918717/176905476-b710e800-1d91-4e87-9480-0385ddf40e3a.png)
面有两个crond进程，其中pid=1118的父进程号是1，即systemd，pid=17687的父进程号是1118，以此可知pid=17687是pid=1118的子进程。

从上面可以看到Linux crond进程是一个daemon进程

 

 

cron的代码结构中，src是核心逻辑。
![image](https://user-images.githubusercontent.com/36918717/176905509-5648c08c-96e9-4a4f-b9bc-b1b2698ce787.png)

main函数在cron.c中，

![image](https://user-images.githubusercontent.com/36918717/176905546-3b5a5049-1e7c-4125-a421-3509e2f5bac9.png)

其中，prefix=NONE

localstatedir='${prefix}/var' ；

sysconfdir='${prefix}/etc'

SPOOL_DIR='${localstatedir}/spool/cron' 即/var/spool/cron

SYS_CROND_DIR='${sysconfdir}/cron.d' 即/etc/cron.d

SYSCRONTAB='${sysconfdir}/crontab' 即/etc/crontab
![image](https://user-images.githubusercontent.com/36918717/176905594-0325fc9f-9f3b-4b5a-a05a-3224f2973be0.png)
主要是这三个目录的文件，然后加载到内存里，每一条crontab执行项都会被fork出一个进程执行。

 

 

 

首先看下load_database实现crontab项加载到内存逻辑
![image](https://user-images.githubusercontent.com/36918717/176905622-98d95aab-2082-49be-a71c-7590e6f9fb09.png)

Load_database主要实现以下逻辑：判断crontab配置目录的mtime是否改变，如果改变则重新加载crontab配置文件，更新内存，否则直接返回。

![image](https://user-images.githubusercontent.com/36918717/176905647-f4997b9c-b976-4619-9e50-fae22093681f.png)
![image](https://user-images.githubusercontent.com/36918717/176905656-243adf51-968d-4dbb-acb2-fff5b49def18.png)

![image](https://user-images.githubusercontent.com/36918717/176905670-99619288-f9b8-4c19-a434-56626c8f915d.png)
比较crontab配置文件所在目录的mtime是否被修改，被修改则重新load对应的crontab内容到内存，否则直接返回，不做任何修改。
![image](https://user-images.githubusercontent.com/36918717/176905687-27258993-64bf-472c-8a23-50982b950b2d.png)

![image](https://user-images.githubusercontent.com/36918717/176905699-ce92436e-8f27-4b4b-b197-879c32c8ebbd.png)

然后执行job_runqueue

![image](https://user-images.githubusercontent.com/36918717/176905723-7189dc4a-aad9-4583-8afe-47ecf17ed93a.png)
![image](https://user-images.githubusercontent.com/36918717/176905731-9f17be1f-18f1-42a1-97ce-1c0d40cd608f.png)

以上就是cron的逻辑。







