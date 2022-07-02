---
layout:     post
title:      linux内核fork剖析
subtitle:   linux内核fork剖析
date:       2020-02-28
author:     lwk
catalog: true
tags:
    - linux
    - fork
---

Linux后台开发的同学应该都使用过多进程或多线程来实现服务的开发，对于使用c/cpp的程序，一般通过fork/pthread_create等方式创建子进程或线程，今天就详细分析fork的具体实现细节。

首先，我们写个简单的例子，通过fork创建子进程，文件名叫p.c吧，代码如下：
```

#include <stdio.h>

#include <unistd.h>

 

int main(int argc, char *argv[])

{

 

       pid_tpid;

       pid= fork();

 

       if(pid < 0){

              printf("errorin fork");

       }else if (pid == 0){

              printf("thisis child process, pid=%d\n", getpid());

       }else {

              printf("thisis parent process, pid=%d\n", getpid());

       }

 

       return0;

}
```
 

编译gcc –g p.c –o p，生成对应的二进制程序p

然后gdb单步执行：

![image](https://user-images.githubusercontent.com/36918717/177003607-41a4ae1f-8edf-435e-b30e-efe19297944b.png)


当想跟踪fork函数体内的时候，出现Missingseparate debuginfos提示，因为没有对应的glibc源码信息，和调试vmcore类似，需要安装对应的debuginfo。

由于yum配置文件默认禁用了从对应的yum源下载debuginfo，所以需要先修改/etc/yum.repos.d/CentOS-Debuginfo.repo里面的debuginfo目录中enabled=1

然后执行debuginfo-install glibc-2.17-292.el7.x86_64

安装完成后，再执行gdb。

![image](https://user-images.githubusercontent.com/36918717/177003616-92c8194d-b54f-4c83-aaa4-915e262435f7.png)
然后打开glibc源码对应的文件sysdeps/nptl/fork.c
![image](https://user-images.githubusercontent.com/36918717/177003622-3f39f9ff-925c-496e-b789-5d4e75c3d2e9.png)
真正执行fork的逻辑函数入口是arch_fork

![image](https://user-images.githubusercontent.com/36918717/177003625-b919dbfa-0645-4e39-9c96-cadd55fba709.png)

![image](https://user-images.githubusercontent.com/36918717/177003632-6f4cce9c-778c-4926-a596-be7187928f8a.png)


这里会从系统调用符号表找到__NR_clone系统调用（具体系统调用可参考之前的文章说一说linux系统调用）。

从上面可以看到arch_fork会通过系统调用最终clone。

 

另外，这里多说一点就是也可以通过strace也能看到系统调用是clone

![image](https://user-images.githubusercontent.com/36918717/177003639-50040dc1-5f5d-40a9-b727-1a1bbe0f3127.png)
![image](https://user-images.githubusercontent.com/36918717/177003641-ee36140b-599f-45bc-92ab-1c1b3e9cddab.png)

好了，知道fork调用的是clone，那么我们找到对应的内核源码吧。

（之前的文章忘记说内核版本了，文章里没有特定说明的话，默认的内核版本是5.5.0）

源码位置kernel/fork.c

![image](https://user-images.githubusercontent.com/36918717/177003642-ec9ad7fa-e180-4304-94d3-0284516af954.png)
最终都是调用_do_fork函数。接下来重点看_do_fork具体做了什么事情
![image](https://user-images.githubusercontent.com/36918717/177003649-8008ab90-aebb-4693-ad2c-85140ac48aa0.png)
其中，copy_process是创建子进程task_struct，并将父进程task_struct数据复制给子进程task_struct

copy_process主要逻辑是：

调用dup_task_struct为新进程创建内核堆栈，包括task_struct和thread_info；

调用copy_creds，为子进程创建新的凭证信息；

调用init_sigpending清除掉挂起的信号；

调用sched_fork()来分割父子进程之间的剩余时间片，将子进程状态置为TASK_NEW；

拷贝进程的所有信息（包括sem、files、fs、sighand、signal、mm、namespaces、io、thread_tls，分别调用copy_semundo()，copy_files(),copy_fs(),copy_sighand(),copy_signal(),copy_mm(),

copy_namespaces(),copy_io(),copy_thread_tls()函数，涉及到的东西比较多，因此就不一一细说了，接下来会以copy_files()为例说下是如何从父进程copy信息的）；

调用alloc_pid，为子进程分配PID；

调用copy_thread_tls，函数里面将寄存器%ax置为0，也是子进程pid返回0的原因

![image](https://user-images.githubusercontent.com/36918717/177003662-51bb02c4-915f-4961-b690-2528448718e8.png)

Copy_process中核心调用之一是dup_task_struct：为子进程分配task_struct和thread_info结构并初始化（用父进程数据进行初始化）

![image](https://user-images.githubusercontent.com/36918717/177003666-cdc983f1-7578-4b5a-986b-e0199ecc521a.png)
dup_task_struct只是对task_struct和thread_info进行分配，并不会分配pid号，pid号是在alloc_pid()函数中进行分配的
![image](https://user-images.githubusercontent.com/36918717/177003673-72450948-38da-4055-b20d-dbcba0f4911e.png)
alloc_pid为子进程分配pid号，分配策略是循环遍历[min,pid_max)，找到没有被使用的pid，pid_max是内核参数，定义在/proc/sys/kernel/pid_max 默认值是32768，如果超过pid_max，即内核查找从min到pid_max没有未使用的pid号时，则内核会报fork error cannot allocate memory。这也是为什么有的时候我们能排查问题时发现系统内存充足，但提示cannot allocate memory的原因。

接下来说下copy_files()函数：
![image](https://user-images.githubusercontent.com/36918717/177003683-74197863-5ca5-44de-b59c-f5682adf7d65.png)

![image](https://user-images.githubusercontent.com/36918717/177003685-f71baa3a-d79d-469c-a746-82a43a700e81.png)

copy_files函数核心掉用是dup_fd():


```

structfiles_struct *dup_fd(struct files_struct *oldf, int *errorp)

{

       struct files_struct *newf;

       struct file **old_fds, **new_fds;

       unsigned int open_files, i;

       struct fdtable *old_fdt, *new_fdt;

 

       *errorp = -ENOMEM;

       newf = kmem_cache_alloc(files_cachep,GFP_KERNEL);//slab内存 分配存储空间

       if (!newf)

              goto out;

 

       atomic_set(&newf->count,1)//files_struct的count设置为1

 

       spin_lock_init(&newf->file_lock);

       newf->resize_in_progress = false;

       init_waitqueue_head(&newf->resize_wait);

       newf->next_fd = 0;

       new_fdt = &newf->fdtab;

       new_fdt->max_fds = NR_OPEN_DEFAULT;//打开最大文件数，默认是64

       new_fdt->close_on_exec =newf->close_on_exec_init;

       new_fdt->open_fds =newf->open_fds_init;

       new_fdt->full_fds_bits =newf->full_fds_bits_init;

       new_fdt->fd =&newf->fd_array[0];

 

       spin_lock(&oldf->file_lock);

       old_fdt = files_fdtable(oldf);//获取父进程的文件描述符信息

       open_files = count_open_files(old_fdt);//统计父进程的文件描述符数

 

       /*

        *Check whether we need to allocate a larger fd array and fd set.

        */

        //内核会检查父进程打开文件的个数。如果父进程打开的文件超过了64个，

        //struct files_struct中自带的数组和位图就不能满足需要了。这种情况下内核会分配一个新的struct fdtable

       while (unlikely(open_files >new_fdt->max_fds)) {

              spin_unlock(&oldf->file_lock);

 

              if (new_fdt !=&newf->fdtab)

                     __free_fdtable(new_fdt);

 

              new_fdt = alloc_fdtable(open_files- 1);//分配新的文件描述符table

              if (!new_fdt) {

                     *errorp = -ENOMEM;

                     goto out_release;

              }

 

              /* beyond sysctl_nr_open; nothingto do */

              // //超出系统单个进程打开的最大文件描述符数fs.nr_open =1048576，则报错返回（注意这里要和file-max区分开，file-max指的是内核可分配的最大文件数）

              if (unlikely(new_fdt->max_fds< open_files)) {

                     __free_fdtable(new_fdt);

                     *errorp = -EMFILE;

                     goto out_release;

              }

 

              /*

               * Reacquire the oldf lock and a pointer to itsfd table

               * who knows it may have a new bigger fd table.We need

               * the latest pointer.

               */

              spin_lock(&oldf->file_lock);

              old_fdt = files_fdtable(oldf);

              open_files = count_open_files(old_fdt);

       }

 

       copy_fd_bitmaps(new_fdt, old_fdt,open_files);//copy父进程的文件bitmap

 

       old_fds = old_fdt->fd;

       new_fds = new_fdt->fd;

 

       //将父进程的open_files个文件描述符copy给子进程

       for (i = open_files; i != 0; i--) {

              struct file *f = *old_fds++;

              if (f) {

                     get_file(f);//f对应的文件的引用计数加1

              } else {

                     /*

                      * The fd may be claimed in the fd bitmap butnot yet

                      * instantiated in the files array if a siblingthread

                      * is partway through open().  So make sure that this

                      * fd is available to the new process.

                      */

                     __clear_open_fd(open_files- i, new_fdt);

              }

              //子进程的struct file类型指针, 指向和父进程相同的struct file 结构体

              rcu_assign_pointer(*new_fds++, f);

       }

       spin_unlock(&oldf->file_lock);

 

       /* clear the remainder */

       memset(new_fds, 0, (new_fdt->max_fds -open_files) * sizeof(struct file *));

 

       rcu_assign_pointer(newf->fdt,new_fdt);

 

       return newf;

 

out_release:

       kmem_cache_free(files_cachep, newf);

out:

       return NULL;

}

 ```

分配完子进程所需要的结构之后，子进程就已经完整的创建出来了，接下来就是调用wake_up_new_task()函数，设置子进程状态为TASK_RUNNING，并将子进程唤醒并放入就绪队列里。

 

以上就是fork的逻辑，从fork具体实现我们可以了解创建一个子进程内核到底做了哪些事情，在我们实际工作中，对同类问题定位有一定的帮助。


























