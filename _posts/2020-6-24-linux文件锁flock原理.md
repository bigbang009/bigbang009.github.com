---
layout:     post
title:      linux文件锁flock原理
subtitle:   linux文件锁flock原理
date:       2020-06-24
author:     lwk
catalog: true
tags:
    - linux
    - kernel
    - flock
---
最近再一次使用到文件锁flock，会使用只是基本技能，那么flock到底是怎么实现的呢？具体原理又是怎样的呢？和以往一样套路-分析源码！

Linux文件锁flock对应的源码是util-linux，git地址https://github.com/karelzak/util-linux

下载下来编译：

./ autogen.sh

./configure

make

编译后在当前目录可以看到生成对应的flock二进制文件。当然还包括其他的二进制，例如dmesg、lscpu、umount、mount等等。

和以往套路一样，gdb 单步走起

![image](https://user-images.githubusercontent.com/36918717/177024051-5f8dd5f0-38be-44f7-a54d-f96ada480789.png)

前面都是一些参数解析操作，所以我们直接b 279行，到flock的系统调用。

fd就是打开文件/tmp/.flock.lock对应的文件描述符；type默认是排它锁(LOCK_EX)，block默认是阻塞式(0)。

Linux系统调用flock我们待会儿再详细分析。

由于默认是阻塞式的，因此如果有进程已经占用了这个文件锁，则flock会一直卡在获取锁。如果进程获取到了文件锁，则不会执行while循环体。继续往下执行
![image](https://user-images.githubusercontent.com/36918717/177024054-0c2ad652-00d0-4ccd-925e-b95769eb662c.png)
![image](https://user-images.githubusercontent.com/36918717/177024057-38d1fef4-907f-4043-866b-6433c5c5d085.png)

上面已经分析完flock的逻辑，下面重点分析下linux系统调用flock的实现。（注意：linux下文件锁flock是util-linux的系统工具，而下面要介绍的flock是linux的系统调用，文件锁flock是通过系统调用flock实现功能的）

flock系统调用定义在fs/locks.c文件中

![image](https://user-images.githubusercontent.com/36918717/177024059-ae0e3bed-283b-4746-9368-34ee51dcf46c.png)
从gdb可以看到，fd=7，type=2，block=0。因此flock系统调用参数fd=7,cmd=2

flock对应的lock类型:
![image](https://user-images.githubusercontent.com/36918717/177024061-db8af4b9-50e5-4b87-96e9-44db6467ebd5.png)

flock_make_lock()函数定义：





