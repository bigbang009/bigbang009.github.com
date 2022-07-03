---
layout:     post
title:      linux下proc文件系统写过程
subtitle:   linux下proc文件系统写过程
date:       2021-07-09
author:     lwk
catalog: true
tags:
    - linux
    - proc文件系统
    - perf
    - 火焰图
---

在之前文件系统系列文章中，阐述过vfs相关知识点，这篇文章通过工具展示虚拟文件系统-proc的读写过程（主要以写为例进行阐述），从而了解proc文件系统和其他文件系统的不同之处。同样，以测试代码为主展开讲解。

测试代码

```
#include <stdio.h>
#include <unistd.h>
#include<stdlib.h>
#include<string.h>
#include<errno.h>
#include<unistd.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>

struct s_test {
    char *a;
};
int main(void)
{


    char buf[100];
    memset(buf,0,sizeof(buf));
    strcpy(buf,"67584");

    while(1) {
        int fd = open("/proc/sys/vm/min_free_kbytes", O_RDWR);//以RW模式打开min_free_kbytes文件
        write(fd,buf,strlen(buf));//向文件写入内容
        close(fd);
        sleep(1);//sleep 1秒
    }
       return 0;
}
```
编译
```
gcc t.c
```
生成二进制a.out

执行./a.out &

获取进程pid
```
ps aux |grep a.out

```
然后使用perf查看调用栈
```
perf record --call-graph dwarf -p 16568
```
间隔10秒之后，停掉perf，然后执行
```
perf report
```
![image](https://user-images.githubusercontent.com/36918717/177037152-42d2570e-c446-48c1-9202-96cae829cda9.png)

根据perf.data生成火焰图：

生成火焰图的工具从github地址https://github.com/brendangregg/FlameGraph下载即可。

执行命令：

```
perf script -i perf.data &> perf.unfold
```
将perf.unfold中的符号进行折叠


```
./stackcollapse-perf.pl perf.unfold &> perf.folded

```
最后生成svg图
```
./flamegraph.pl perf.folded > perf.svg

```
生成火焰图看更直观一些


![image](https://user-images.githubusercontent.com/36918717/177037172-1f886757-c6d6-467a-bc68-50aa7c0b9e03.png)
红色箭头所指的就是写/proc/sys/vm/min_free_kbytes的整个内核过程。

接下来根据源码看下内核proc系统文件的整个详细过程。

在之前的vfs文章中分析过vfs最终读写到具体的文件系统时会执行具体文件系统的file_operations，例如xfs，ext4等，都有自己特有的file_operations。同样，针对proc文件系统，也有自己的file_operations，那就是fs/proc/proc_sysctl.c的proc_sys_file_operations
```
static const struct file_operations proc_sys_file_operations = {
    .open       = proc_sys_open,
    .poll       = proc_sys_poll,
    .read       = proc_sys_read,
    .write      = proc_sys_write,
    .llseek     = default_llseek,
};
```
因此针对proc文件系统的write，执行的是proc_sys_write()函数。

顺着proc_sys_write()函数继续往下找

```
static ssize_t proc_sys_write(struct file *filp, const char __user *buf,
                size_t count, loff_t *ppos)
{
    return proc_sys_call_handler(filp, (void __user *)buf, count, ppos, 1);
}


static ssize_t proc_sys_call_handler(struct file *filp, void __user *buf,
    size_t count, loff_t *ppos, int write)
{
  struct inode *inode = file_inode(filp);//获取文件inode信息
  struct ctl_table_header *head = grab_header(inode);
  struct ctl_table *table = PROC_I(inode)->sysctl_entry;//将inode转成proc_inode
  void *new_buf = NULL;
  ssize_t error;

  if (IS_ERR(head))
    return PTR_ERR(head);

  /*
   * At this point we know that the sysctl was not unregistered
   * and won't be until we finish.
   */
  error = -EPERM;
  if (sysctl_perm(head, table, write ? MAY_WRITE : MAY_READ))//判断文件读写类型
    goto out;

  /* if that can happen at all, it should be -EINVAL, not -EISDIR */
  error = -EINVAL;
  if (!table->proc_handler)//没有回调函数
    goto out;

  error = BPF_CGROUP_RUN_PROG_SYSCTL(head, table, write, buf, &count,
             ppos, &new_buf);
  if (error)
    goto out;

  /* careful: calling conventions are nasty here */
  if (new_buf) {
    mm_segment_t old_fs;

    old_fs = get_fs();
    set_fs(KERNEL_DS);
    error = table->proc_handler(table, write, (void __user *)new_buf,
              &count, ppos);
    set_fs(old_fs);
    kfree(new_buf);
  } else {
    error = table->proc_handler(table, write, buf, &count, ppos);//执行回调函数
  }

  if (!error)
    error = count;
out:
  sysctl_head_finish(head);

  return error;
}

```
那proc_handler又是什么呢？是怎么来的呢？

proc_handler()是个回调函数，属于struct ctl_table结构里的。
```
/* A sysctl table is an array of struct ctl_table: */
struct ctl_table {
  const char *procname;    /* Text ID for /proc/sys, or zero */
  void *data;
  int maxlen;
  umode_t mode;
  struct ctl_table *child;  /* Deprecated */
  proc_handler *proc_handler;  /* Callback for text formatting */
  struct ctl_table_poll *poll;
  void *extra1;
  void *extra2;
} __randomize_layout;

```
proc_handler主要是内核参数注册的回调函数，内核初始化时，会执行sysctl_init()函数

```
int __init sysctl_init(void)
{
    struct ctl_table_header *hdr;

    hdr = register_sysctl_table(sysctl_base_table);
    kmemleak_not_leak(hdr);
    return 0;
}
```
而sysctl_base_table是定义了内核参数的一个struct ctl_table类型的数组，调用register_sysctl_table(sysctl_base_table)对每个struct ctl_table成员进行注册。也就是每个内核参数ctl_table进行注册（包括procname，proc_handler）
```
struct ctl_table_header *register_sysctl_table(struct ctl_table *table)
{
    static const struct ctl_path null_path[] = { {} };

    return register_sysctl_paths(null_path, table);
}
EXPORT_SYMBOL(register_sysctl_table);

    {    
        .procname   = "min_free_kbytes",
        .data       = &min_free_kbytes,
        .maxlen     = sizeof(min_free_kbytes),
        .mode       = 0644,
        .proc_handler   = min_free_kbytes_sysctl_handler,
        .extra1     = SYSCTL_ZERO,
}

```
![image](https://user-images.githubusercontent.com/36918717/177037226-923cd56c-5db8-465a-9ab4-1a1a122c5a10.png)
因此table->proc_handler最终调用的是min_free_kbytes_sysctl_handler()，这也和火焰图上对得上。

接下来看下min_free_kbytes_sysctl_handler()函数

该函数定义在mm/page_alloc.c中。
```
int min_free_kbytes_sysctl_handler(struct ctl_table *table, int write,
    void __user *buffer, size_t *length, loff_t *ppos)
{
    int rc;

    rc = proc_dointvec_minmax(table, write, buffer, length, ppos);
    if (rc) 
        return rc;

    if (write) {
        user_min_free_kbytes = min_free_kbytes;
        setup_per_zone_wmarks();
    }    
    return 0;
}
```
其中setup_per_zone_wmarks()在上一章聊聊linux内存的watermark已经详细阐述过。

proc_dointvec_minmax()函数最终实现将用户空间的min_free_kbytes拷贝到内核空间
![image](https://user-images.githubusercontent.com/36918717/177037240-964649b9-d91a-4fbd-8b73-ec5cba07b5b3.png)

其中copy_from_user不在本章的讨论范围，因此先不做阐述。












