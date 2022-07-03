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

![image](https://user-images.githubusercontent.com/36918717/177027555-744e237d-a54b-4f05-9cf6-66f7c799927a.png)

![image](https://user-images.githubusercontent.com/36918717/177027560-5c1d4990-b4fc-4551-a36f-8d5215905eef.png)
由于参数fl是NULL，因此会调用locks_alloc_lock()函数分配file_lock，locks_alloc_lock()调用locks_init_lock_heads()对file_lock进行初始化。

![image](https://user-images.githubusercontent.com/36918717/177027563-697c0903-7784-4c8e-84bc-67197cceb50b.png)

看下file_lock的结构已经核心字段的意思。
![image](https://user-images.githubusercontent.com/36918717/177027568-951b6b7a-cd39-4cfa-b1d2-c0836116cb16.png)

![image](https://user-images.githubusercontent.com/36918717/177027569-7ab3fc52-f999-4f19-81ba-853ea83c8a98.png)
flock定义在fs/fuse/file.c文件中（具体根据文件系统类型）

![image](https://user-images.githubusercontent.com/36918717/177027571-90c36441-68d5-4c52-a85a-75ce125db0fa.png)

看下fuse_file_flock()函数实现：
![image](https://user-images.githubusercontent.com/36918717/177027573-29544883-f9ec-42bc-85f2-e7466827a6bb.png)
看下locks_lock_file_wait()函数，主要是等获取锁。

![image](https://user-images.githubusercontent.com/36918717/177027575-76e212c8-1a13-4a87-98b5-3b80a6dff31b.png)
![image](https://user-images.githubusercontent.com/36918717/177027576-f8747b70-1b3c-4478-9a38-2cf4325da5fc.png)

![image](https://user-images.githubusercontent.com/36918717/177027579-725792f6-e2db-439a-9cea-a646b9893407.png)
如果锁被占用，则flock_lock_inode_wait()就会循环等待获取锁。


