---
layout:     post
title:      linux内核VFS之open
subtitle:   linux内核VFS之open
date:       2021-01-11
author:     lwk
catalog: true
tags:
    - linux
    - vfs
    - open
---
很久没有写文章了，这是2021年的第一篇文章，主题还是linux内核相关的，接着之前的VFS继续介绍open相关内容。内容比较简单，没有很详细&深入介绍每一个内容。

Linux的open也是通过系统调用进入到内核态，其中内核里的定义入口文件在fs/open.c，具体如下：
```

SYSCALL_DEFINE3(open, const char __user *,filename, int, flags, umode_t, mode)
{
       if(force_o_largefile())
              flags|= O_LARGEFILE;
 
       returndo_sys_open(AT_FDCWD, filename, flags, mode);
}
```
```
long do_sys_open(int dfd, const char __user*filename, int flags, umode_t mode)
{
       structopen_flags op;
       intfd = build_open_flags(flags, mode, &op);//根据flags、mode设置op的属性
       structfilename *tmp;
 
       if(fd) // build_open_flags永远返回是0，所以if永远是false
              returnfd;
 
       tmp= getname(filename);
       if(IS_ERR(tmp))
              returnPTR_ERR(tmp);
 
       fd= get_unused_fd_flags(flags); // 分配未使用的fd
       if(fd >= 0) {
              structfile *f = do_filp_open(dfd, tmp, &op);//打开文件，获取file结构体，文件操作的函数指针等
              if(IS_ERR(f)) {
                     put_unused_fd(fd);
                     fd= PTR_ERR(f);
              }else {
                     fsnotify_open(f);
                     fd_install(fd,f);//文件描述符与文件结构体(真正的文件操作函数等)关联
              }
       }
       putname(tmp);
       returnfd;
}
```
```

int get_unused_fd_flags(unsigned flags)
{
       return__alloc_fd(current->files, 0, rlimit(RLIMIT_NOFILE), flags);
}
```
```

int __alloc_fd(struct files_struct *files,
              unsigned start, unsigned end, unsignedflags)
{
       unsignedint fd;
       interror;
       structfdtable *fdt;
 
       spin_lock(&files->file_lock);
repeat:
       fdt= files_fdtable(files);
       fd= start;
       if(fd < files->next_fd) // 下一个将被分配的文件描述符号
              fd= files->next_fd;
 
       if(fd < fdt->max_fds)
              fd= find_next_fd(fdt, fd); // 查找下一个未被使用的文件描述符
 
       /*
        * N.B. For clone tasks sharing a filesstructure, this test
        * will limit the total number of files thatcan be opened.
        */
       error= -EMFILE;
       if(fd >= end)
              gotoout;
 
       error= expand_files(files, fd);//扩展文件描述符表（因为刚开始系统没有分配sysctl_nr_open的文件描述符表，因为虽然设置了最大使用的文件描述符表，
                                                               //但可能进程根本就用不到这么，所以，为了节省内存，刚开始的文件描述符表比较小，当达到了限制后，再根据实际情况
                                                               //进行扩展（若满足扩展条件，则内核进行扩展，但最大也不会超过sysctl_nr_open,超过则报错）
       if(error < 0)
              gotoout;
 
       /*
        * If we needed to expand the fs array we
        * might have blocked - try again.
        */
       if(error)
              gotorepeat;
 
       if(start <= files->next_fd)
              files->next_fd= fd + 1;
 
       __set_open_fd(fd,fdt);
       if(flags & O_CLOEXEC)
              __set_close_on_exec(fd,fdt);
       else
              __clear_close_on_exec(fd,fdt);
       error= fd;
#if 1
       /*Sanity check */
       if(rcu_access_pointer(fdt->fd[fd]) != NULL) {
              printk(KERN_WARNING"alloc_fd: slot %d not NULL!\n", fd);
              rcu_assign_pointer(fdt->fd[fd],NULL);
       }
#endif
 
out:
       spin_unlock(&files->file_lock);
       returnerror;
}
```
find_next_fd()函数主要是查找下一个未被使用的文件描述符(unsigned int类型)，并分给待打开的文件作为fd标识。内核使用文件描述符表表示可以分配的描述符信息。在前面的文章

我们看到进程里打开文件的结构表示：
```

struct files_struct {
  /*
   *read mostly part
   */
       atomic_tcount;// 共享该表的进程数目
       boolresize_in_progress;
       wait_queue_head_tresize_wait;
 
       structfdtable __rcu *fdt;
       structfdtable fdtab; //文件描述符表
  /*
   *written part on a separate cache line in SMP
   */
       spinlock_tfile_lock ____cacheline_aligned_in_smp;
       unsignedint next_fd;//所分配的最大文件描述符加1
       unsignedlong close_on_exec_init[1];//执行exec() 时需要关闭的文件描述符的集合
       unsignedlong open_fds_init[1];//文件描述符的初值集合
       unsignedlong full_fds_bits_init[1];
       structfile __rcu * fd_array[NR_OPEN_DEFAULT];//文件对象指针的初始化数组
};
```
其中fdt就是文件描述符表：

```

struct fdtable {
       unsignedint max_fds;
       structfile __rcu **fd; //打开的文件描述符指针数组     /* current fd array */
       unsignedlong *close_on_exec;
       unsignedlong *open_fds;//指向打开的文件描述符指针
       //full_fds_bits每1bit代表的是一个64位的数组，也就是说代表了64位描述符；
       //内核中的位图是一片连续的内存空间，最低bit表示数值0，下一比特表示1，依次类推；
       //full_fds_bits每1bit只有0和1两个值，0表示有该组有可用的文件描述符，1表示没有可用的文件描述符，
       //例如位图bit 0代表的是0-63共64个文件描述符，bit1代表的是64-127共64个文件描述符，假如0-63文件描述符都被使用了，
       //那么位图bit0则应该标记为1，如果64-127中有一个未使用的文件描述符，则bit1被标记为0，当64-127中的所有文件描述符都被使用的时候，才标记为1。
       unsignedlong *full_fds_bits;
       structrcu_head rcu;
};
```
```

static unsigned int find_next_fd(structfdtable *fdt, unsigned int start)
{
       unsignedint maxfd = fdt->max_fds; // 文件描述符表最大文件描述符
       unsignedint maxbit = maxfd / BITS_PER_LONG; // 最大文件描述符组(一组文件描述符包含64个文件描述符，例如0-63为一组)
       unsignedint bitbit = start / BITS_PER_LONG; // 第几组文件描述符，从0开始。例如起始查找文件描述符所在组(64个文件描述符为一组，我们要从文件描述符65开始查找，可知，65文件描述符在65/64 = 1组，因此我们从第1组开始查找即可)
 
       bitbit= find_next_zero_bit(fdt->full_fds_bits, maxbit, bitbit) * BITS_PER_LONG;//  查找下一个可用文件描述符组，结果乘以BITS_PER_LONG，即得到该组起始文件描述符
       if(bitbit > maxfd)
              returnmaxfd; // 可用文件描述符起始值大于最大文件描述符，直接返回最大文件描述符，表示文件描述符需要扩展。
       if(bitbit > start)
              start= bitbit; // 可用起始文件描述符大于参数传递的起始查找文件描述符，将开始查找的值从真正有效的值开始，避免做无效的查找
       returnfind_next_zero_bit(fdt->open_fds, maxfd, start); // 从文件描述符表中查找可用的文件描述符(之前是查找可用组，以64位为大小查找，提高效率，这次的查找范围缩小的组内了，即最多只需要查找64次了)。
}
```
expand_files()函数主要是做文件描述符扩展使用，具体如下：


```
static int expand_files(struct files_struct*files, unsigned int nr)
       __releases(files->file_lock)
       __acquires(files->file_lock)
{
       structfdtable *fdt;
       intexpanded = 0;
 
repeat:
       fdt= files_fdtable(files);//获取当前的fdt
 
       /*Do we need to expand? */
       if(nr < fdt->max_fds)
              returnexpanded;
 
       /*Can we expand? */
       if(nr >= sysctl_nr_open) // 大于当前系统的最大文件描述符数
              return-EMFILE;
 
       if(unlikely(files->resize_in_progress)) {
              spin_unlock(&files->file_lock);
              expanded= 1;
              wait_event(files->resize_wait,!files->resize_in_progress);
              spin_lock(&files->file_lock);
              gotorepeat;
       }
 
       /*All good, so we try */
       files->resize_in_progress= true;
       expanded= expand_fdtable(files, nr); // 扩展文件描述符表
}
```
expand_fdtable()函数如下：


```
static int expand_fdtable(structfiles_struct *files, unsigned int nr)
       __releases(files->file_lock)
       __acquires(files->file_lock)
{
       structfdtable *new_fdt, *cur_fdt;
 
       spin_unlock(&files->file_lock);
       new_fdt= alloc_fdtable(nr);//分配新的fdt
 
       /*make sure all __fd_install() have seen resize_in_progress
        * or have finished their rcu_read_lock_sched()section.
        */
       if(atomic_read(&files->count) > 1)
              synchronize_rcu();
 
       spin_lock(&files->file_lock);
       if(!new_fdt)
              return-ENOMEM;
       /*
        * extremely unlikely race - sysctl_nr_opendecreased between the check in
        * caller and alloc_fdtable().  Cheaper to catch it here...
        */
       if(unlikely(new_fdt->max_fds <= nr)) {
              __free_fdtable(new_fdt);
              return-EMFILE;
       }
       cur_fdt= files_fdtable(files);//获取旧的fdt
       BUG_ON(nr< cur_fdt->max_fds);
       copy_fdtable(new_fdt,cur_fdt); // 将旧的fdt拷贝到新的fdt
       rcu_assign_pointer(files->fdt,new_fdt);
       if(cur_fdt != &files->fdtab)
              call_rcu(&cur_fdt->rcu,free_fdtable_rcu); // 释放旧的fdt
       /*coupled with smp_rmb() in __fd_install() */
       smp_wmb();
       return1;
}
```
其中fdt是使用bit map表示的，如下图所示：
![image](https://user-images.githubusercontent.com/36918717/177033686-04127180-21af-4bf1-8097-252fe9d7a181.png)
full_fds_bits是0~N，full_fds_bits中bit对应64个bit位（64位操作系统），bit位如果是1表示对应的fd已被占用，为0表示未被使用。系统启动时，默认给fdt->max_fds分配的是64个可用文件句柄（0-63），即full_fds_bits的位数是1，对应的分组号是0，则fd[0]指向64bit的连续内存空间的数组。
上面详细分析了get_unused_fd_flags()函数是如何分配文件描述符的，接下来重点看下do_filp_open()函数。
```
struct file *do_filp_open(int dfd, structfilename *pathname,
              conststruct open_flags *op)
{
       structnameidata nd;
       intflags = op->lookup_flags;
       structfile *filp;
 
       set_nameidata(&nd,dfd, pathname);
       filp= path_openat(&nd, op, flags | LOOKUP_RCU);//打开文件
       if(unlikely(filp == ERR_PTR(-ECHILD)))
              filp= path_openat(&nd, op, flags);
       if(unlikely(filp == ERR_PTR(-ESTALE)))
              filp= path_openat(&nd, op, flags | LOOKUP_REVAL);
       restore_nameidata();
       returnfilp;
}
```
```

static struct file *path_openat(structnameidata *nd,
                     conststruct open_flags *op, unsigned flags)
{
       structfile *file;
       interror;
 
       file= alloc_empty_file(op->open_flag, current_cred());//创建struct file，主要是分配内存；current_cred()当前进程的凭证信息(例如SYS_ADMIN)
       if(IS_ERR(file))
              returnfile;
 
       if(unlikely(file->f_flags & __O_TMPFILE)) {
              error= do_tmpfile(nd, flags, op, file);
       }else if (unlikely(file->f_flags & O_PATH)) {
              error= do_o_path(nd, flags, file);
       }else {
              constchar *s = path_init(nd, flags);// path_init为nd的path和inode赋值，为path->dentry赋值. static int link_path_walk 为nd->last 赋值
              while(!(error = link_path_walk(s, nd)) &&
                     (error= do_last(nd, file, op)) > 0) { // link_path_walk一级一级解析文件路径
                     nd->flags&= ~(LOOKUP_OPEN|LOOKUP_CREATE|LOOKUP_EXCL);
                     s= trailing_symlink(nd);
              }
              terminate_walk(nd);
       }
       fput(file);
}
```
// set_nameidata主要是保护当前进程的现场信息，主要是
```
static void set_nameidata(struct nameidata*p, int dfd, struct filename *name)
{
       structnameidata *old = current->nameidata;
       p->stack= p->internal;
       p->dfd= dfd;
       p->name= name;
       p->total_link_count= old ? old->total_link_count : 0;
       p->saved= old;
       current->nameidata= p;
}
```
```
struct nameidata {

       structpath     path; //记录路径查找的结果

       structqstr      last; //路径名最后一个分量

       structpath     root; //路径查找的根目录信息

       structinode   *inode; /* path.dentry.d_inode */

       unsignedint   flags;

       unsignedseq, m_seq;

       int          last_type; //记录当前查找到目录的类别Normal/Dot/DotDot/Root/Bind

       unsigneddepth; //查找过程中跨越的符号链接深度

       int          total_link_count; //查找过程中经过的符号链接总数

       structsaved {

              structpath link;

              structdelayed_call done;

              constchar *name;

              unsignedseq;

       }*stack, internal[EMBEDDED_LEVELS];//用来记录查找过程中碰到的符号链接

       structfilename      *name;

       structnameidata *saved;

       structinode   *link_inode;

       unsignedroot_seq;

       int          dfd;

} __randomize_layout;
```
```

struct file *alloc_empty_file(int flags,const struct cred *cred)

{

       staticlong old_max;

       structfile *f;

 

       /*

        * Privileged users can go above max_files

        */

        //get_nr_files函数返回percpu_counter_read_positive(&nr_files)的值

       if(get_nr_files() >= files_stat.max_files && !capable(CAP_SYS_ADMIN)){ //超过最大文件数限制且不具备CAP_SYS_ADMIN权限

              /*

               * percpu_counters are inaccurate.  Do an expensive check before

               * we go and fail.

               */

              if(percpu_counter_sum_positive(&nr_files) >= files_stat.max_files)

                     gotoover;

       }

 

       f= __alloc_file(flags, cred);//分配文件对象

       if(!IS_ERR(f))

              percpu_counter_inc(&nr_files);

 

       returnf;

 

over:

       /*Ran out of filps - report that */

       if(get_nr_files() > old_max) {

              pr_info("VFS:file-max limit %lu reached\n", get_max_files());

              old_max= get_nr_files();

       }

       returnERR_PTR(-ENFILE);

}
```
```

static struct file *__alloc_file(int flags,const struct cred *cred)

{

       structfile *f;

       interror;

 

       f= kmem_cache_zalloc(filp_cachep, GFP_KERNEL);//分配文件对象所用内存并清零

       if(unlikely(!f))

              returnERR_PTR(-ENOMEM);

 

       f->f_cred= get_cred(cred);//获取凭证

       error= security_file_alloc(f);

       if(unlikely(error)) {

              file_free_rcu(&f->f_u.fu_rcuhead);

              returnERR_PTR(error);

       }

 

       atomic_long_set(&f->f_count,1);

       rwlock_init(&f->f_owner.lock);

       spin_lock_init(&f->f_lock);

       mutex_init(&f->f_pos_lock);

       eventpoll_init_file(f);

       f->f_flags= flags;

       f->f_mode= OPEN_FMODE(flags);

       /*f->f_version: 0 */

 

       returnf;

}
```
kmem_cache_zalloc()会调用kmem_cache_alloc()，kmem_cache_alloc()又调用slab_alloc(),slab_alloc()函数再调用slab_alloc_node()，slab_alloc_node()函数在之前的文章kmalloc机制有说过，主要是slub(slab)内存分配机制，这里不再赘述。

do_last()函数
```
int vfs_open(const struct path *path,struct file *file)

{

       file->f_path= *path;

       returndo_dentry_open(file, d_backing_inode(path->dentry), NULL);//do_dentry_open会调用file对应的文件系统的file_operations（具体文件系统的open）

}
```
关于文件系统file_operations，在之前的文件VFS系统篇有阐述过，这里不再赘述。

最后，有了fd和file之后，接下来执行fd和file的关联操作。
```
void fd_install(unsigned int fd, structfile *file)
{

       __fd_install(current->files,fd, file);

}
```
__fd_install()函数执行将当前进程的fdt[fd]指向file，这样fd和file就关联起来了。

 

以上就是linux kernel vfs的open过程














