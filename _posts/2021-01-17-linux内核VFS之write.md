---
layout:     post
title:      linux内核VFS之write
subtitle:   linux内核VFS之write
date:       2021-01-17
author:     lwk
catalog: true
tags:
    - linux
    - vfs
    - write
---

今天主要看下VFS的write()函数逻辑是怎么实现的，write()函数在内核里的定义入口文件在fs/read_write.c，具体如下：

```
SYSCALL_DEFINE3(write, unsigned int, fd,const char __user *, buf,
              size_t,count)
{
       returnksys_write(fd, buf, count);
}
 
ssize_t ksys_write(unsigned int fd, constchar __user *buf, size_t count)
{
       structfd f = fdget_pos(fd); // 根据文件描述符返回对应的file信息(struct fd包含了file)
       ssize_tret = -EBADF;
 
       if(f.file) {
              loff_tpos, *ppos = file_ppos(f.file);
              if(ppos) {
                     pos= *ppos;
                     ppos= &pos;
              }
              ret= vfs_write(f.file, buf, count, ppos);//写文件
              if(ret >= 0 && ppos)
                     f.file->f_pos= pos;
              fdput_pos(f);
       }
 
       returnret;
}
```
接下来重点看下vfs_write()函数：
```
ssize_t vfs_write(struct file *file, constchar __user *buf, size_t count, loff_t *pos)
{
       ssize_tret;
 
       if(!(file->f_mode & FMODE_WRITE)) // 检查文件mode
              return-EBADF;
       if(!(file->f_mode & FMODE_CAN_WRITE))
              return-EINVAL;
       if(unlikely(!access_ok(buf, count)))
              return-EFAULT;
 
       ret= rw_verify_area(WRITE, file, pos, count);//对文件锁的校验(主要包括冲突、死锁校验)
       if(!ret) {
              if(count > MAX_RW_COUNT)
                     count=  MAX_RW_COUNT;
              file_start_write(file);//判断文件系统等级（未冻结、冻结写、冻结页缺失、冻结、完全冻结），并将写者加一
              ret= __vfs_write(file, buf, count, pos); //调用对应文件系统执行写操作
              if(ret > 0) {
                     fsnotify_modify(file);
                     add_wchar(current,ret);
              }
              inc_syscw(current);//当前进程的系统调+1
              file_end_write(file);
       }
 
       returnret;
}
```

rw_verify_area()函数主要对待写区域进行检查，包括加锁、死锁检测等；__vfs_write()会调用对应file的wirte操作，wirte操作是在文件系统初始化的时候设置的回调函数，针对不同文件系统在初始化的设置会设置对应的回调函数，例如xfs，对应的回调函数在文件fs/xfs/xfs_file.c：
```

const struct file_operationsxfs_file_operations = {
       .llseek            = xfs_file_llseek,
       .read_iter= xfs_file_read_iter,
       .write_iter       = xfs_file_write_iter,
       .splice_read    = generic_file_splice_read,
       .splice_write   = iter_file_splice_write,
       .iopoll            = iomap_dio_iopoll,
       .unlocked_ioctl      = xfs_file_ioctl,
#ifdef CONFIG_COMPAT
       .compat_ioctl  = xfs_file_compat_ioctl,
#endif
       .mmap           = xfs_file_mmap,
       .mmap_supported_flags= MAP_SYNC,
       .open             = xfs_file_open,
       .release   = xfs_file_release,
       .fsync             = xfs_file_fsync,
       .get_unmapped_area= thp_get_unmapped_area,
       .fallocate= xfs_file_fallocate,
       .fadvise   = xfs_file_fadvise,
       .remap_file_range= xfs_file_remap_range,
};
```

其实，我在之前的文章linux内核VFS之read有使用xfs的read进行阐述，这次write也使用xfs进行阐述。

 

下面首先详细介绍函数rw_verify_area():
```

int rw_verify_area(int read_write, structfile *file, const loff_t *ppos, size_t count)
{
       structinode *inode;
       intretval = -EINVAL;
 
       inode= file_inode(file);//返回struct file对应的struct inode
       if(unlikely((ssize_t) count < 0))
              returnretval;
 
       /*
        * ranged mandatory locking does not apply tostreams - it makes sense
        * only for files where position has a meaning.
        */
       if(ppos) {
              loff_tpos = *ppos;
 
              if(unlikely(pos < 0)) {
                     if(!unsigned_offsets(file))
                            returnretval;
                     if(count >= -pos) /* both values are in 0..LLONG_MAX */
                            return-EOVERFLOW;
              }else if (unlikely((loff_t) (pos + count) < 0)) {
                     if(!unsigned_offsets(file))
                            returnretval;
              }
 
              if(unlikely(inode->i_flctx && mandatory_lock(inode))) {
                     retval= locks_mandatory_area(inode, file, pos, pos + count - 1,
                                   read_write== READ ? F_RDLCK : F_WRLCK);//对待操作区域[pos,pos+count-1]进行加锁
                     if(retval < 0)
                            returnretval;
              }
       }
 
       returnsecurity_file_permission(file,
                            read_write== READ ? MAY_READ : MAY_WRITE);
}
```
重点看locks_mandatory_area()函数是如何对待操作区域进行加锁的。


```
int locks_mandatory_area(struct inode*inode, struct file *filp, loff_t start,
                      loff_t end, unsigned char type)
{
       structfile_lock fl;
       interror;
       boolsleep = false;
 
       locks_init_lock(&fl);// 初始化file_lock
       fl.fl_pid= current->tgid;//当前线程id
       fl.fl_file= filp;
       fl.fl_flags= FL_POSIX | FL_ACCESS;
       if(filp && !(filp->f_flags & O_NONBLOCK)) // 是否非阻塞模式
              sleep= true; //阻塞模式
       fl.fl_type= type; // type=F_WRLCK=2
       fl.fl_start= start; // 被锁区域的开始位置
       fl.fl_end= end; // 被锁区域的结束位置
 
       for(;;) {
              if(filp) {
                     fl.fl_owner= filp;
                     fl.fl_flags&= ~FL_SLEEP;
                     error= posix_lock_inode(inode, &fl, NULL);
                     if(!error)
                            break;
              }
 
              if(sleep)
                     fl.fl_flags|= FL_SLEEP; //阻塞模式，fl_flags增加FL_SLEEP标识
              fl.fl_owner= current->files;//拥有锁标识的files_struct
              error= posix_lock_inode(inode, &fl, NULL);
              if(error != FILE_LOCK_DEFERRED)
                     break;
              error= wait_event_interruptible(fl.fl_wait, !fl.fl_blocker);
              if(!error) {
                     /*
                      * If we've been sleeping someone might have
                      * changed the permissions behind our back.
                      */
                     if(__mandatory_lock(inode))
                            continue;
              }
 
              break;
       }
       locks_delete_block(&fl);
 
       returnerror;
}
 
static int posix_lock_inode(struct inode*inode, struct file_lock *request,
                         struct file_lock *conflock)
{
       structfile_lock *fl, *tmp;
       structfile_lock *new_fl = NULL;
       structfile_lock *new_fl2 = NULL;
       structfile_lock *left = NULL;
       structfile_lock *right = NULL;
       structfile_lock_context *ctx;
       interror;
       booladded = false;
       LIST_HEAD(dispose);
 
       //分配file_lock_context
       ctx= locks_get_lock_context(inode, request->fl_type); //request->fl_type=2
       if(!ctx)
              return(request->fl_type == F_UNLCK) ? 0 : -ENOMEM;
 
       /*
        * We may need two file_lock structures forthis operation,
        * so we get them in advance to avoid races.
        *
        * In some cases we can be sure, that no newlocks will be needed
        */
       if(!(request->fl_flags & FL_ACCESS) &&
           (request->fl_type != F_UNLCK ||
            request->fl_start != 0 ||request->fl_end != OFFSET_MAX)) {
              new_fl= locks_alloc_lock();
              new_fl2= locks_alloc_lock();
       }
 
       percpu_down_read(&file_rwsem);
       spin_lock(&ctx->flc_lock);
       /*
        * New lock request. Walk all POSIX locks andlook for conflicts. If
        * there are any, either return error or putthe request on the
        * blocker's list of waiters and the globalblocked_hash.
        */
       if(request->fl_type != F_UNLCK) {
              list_for_each_entry(fl,&ctx->flc_posix, fl_list) {
                     if(!posix_locks_conflict(request, fl))//检查request和已有的链表中的file_lock是否有冲突
                            continue;//没冲突则继续遍历
                     if(conflock)
                            locks_copy_conflock(conflock,fl);
                     error= -EAGAIN;
                     if(!(request->fl_flags & FL_SLEEP))
                            gotoout;
                     /*
                      * Deadlock detection and insertion into theblocked
                      * locks list must be done while holding thesame lock!
                      */
                     error= -EDEADLK;
                     spin_lock(&blocked_lock_lock);
                     /*
                      * Ensure that we don't find any locks blockedon this
                      * request during deadlock detection.
                      */
                     __locks_wake_up_blocks(request);
                     if(likely(!posix_locks_deadlock(request, fl))) {
                            error= FILE_LOCK_DEFERRED;
                            __locks_insert_block(fl,request,
                                               posix_locks_conflict);
                     }
                     spin_unlock(&blocked_lock_lock);
                     gotoout;
              }
       }
 
       /*If we're just looking for a conflict, we're done. */
       error= 0;
       if(request->fl_flags & FL_ACCESS)//如果fl_flags按位与not trying to lock, just looking 则转到:out
              gotoout;
 
       /*Find the first old lock with the same owner as the new lock */
       list_for_each_entry(fl,&ctx->flc_posix, fl_list) {
              if(posix_same_owner(request, fl))//检查两个锁的owner是否是同一个
                     break;
       }
 
       /*Process locks with this owner. */
       list_for_each_entry_safe_from(fl,tmp, &ctx->flc_posix, fl_list) {
              if(!posix_same_owner(request, fl))//两个锁的owner是否是同一个
                     break;//不是同一个则跳出
 
              /*Detect adjacent or overlapping regions (if same lock type) */
              if(request->fl_type == fl->fl_type) { //owner一致且锁类型也一致
                     /*In all comparisons of start vs end, use
                      * "start - 1" rather than "end+ 1". If end
                      * is OFFSET_MAX, end + 1 will become negative.
                      */
                     if(fl->fl_end < request->fl_start - 1)
                            continue;
                     /*If the next lock in the list has entirely bigger
                      * addresses than the new one, insert the lockhere.
                      */
                     if(fl->fl_start - 1 > request->fl_end)
                            break;
 
                     /*If we come here, the new and old lock are of the
                      * same type and adjacent or overlapping. Makeone
                      * lock yielding from the lower start addressof both
                      * locks to the higher end address.
                      */
                     //owner一致且锁类型一致，两个锁的覆盖范围又有重叠，则选择start最小的作为起始位置；选择end最大的作为结束位置
                     if(fl->fl_start > request->fl_start)
                            fl->fl_start= request->fl_start;
                     else
                            request->fl_start= fl->fl_start;
                     if(fl->fl_end < request->fl_end)
                            fl->fl_end= request->fl_end;
                     else
                            request->fl_end= fl->fl_end;
                     if(added) {
                            locks_delete_lock_ctx(fl,&dispose);
                            continue;
                     }
                     request= fl;
                     added= true;
              }else { // 处理两个锁类型不一致的情况
                     /*Processing for different lock types is a bit
                      * more complex.
                      */
                     if(fl->fl_end < request->fl_start)//无重叠区域，且request锁住区域在f1区域之前
                            continue;
                     if(fl->fl_start > request->fl_end)//无重叠区域，且request锁住区域在f1区域之后
                            break;
                     if(request->fl_type == F_UNLCK)
                            added= true;
                     if(fl->fl_start < request->fl_start)//有重叠区域，且request锁住区域的起始位置在f1锁住区域之后
                            left= fl;
                     /*If the next lock in the list has a higher end
                      * address than the new one, insert the new onehere.
                      */
                     if(fl->fl_end > request->fl_end) {//有重叠区域，且request锁住区域的结束位置在f1锁住区域之前
                            right= fl;
                            break;
                     }
                     if(fl->fl_start >= request->fl_start) {
                            /*The new lock completely replaces an old
                             * one (This may happen several times).
                             */
                            if(added) {
                                   locks_delete_lock_ctx(fl,&dispose);
                                   continue;
                            }
                            /*
                             * Replace the old lock with new_fl, and
                             * remove the old one. It's safe to do the
                             * insert here since we know that we won't be
                             * using new_fl later, and that the lock is
                             * just replacing an existing lock.
                             */
                            error= -ENOLCK;
                            if(!new_fl)
                                   gotoout;
                            locks_copy_lock(new_fl,request);
                            request= new_fl;
                            new_fl= NULL;
                            locks_insert_lock_ctx(request,&fl->fl_list);
                            locks_delete_lock_ctx(fl,&dispose);
                            added= true;
                     }
              }
       }
 
       /*
        * The above code only modifies existing locksin case of merging or
        * replacing. If new lock(s) need to beinserted all modifications are
        * done below this, so it's safe yet to bailout.
        */
       error= -ENOLCK; /* "no luck" */
       if(right && left == right && !new_fl2)
              gotoout;
 
       error= 0;
       if(!added) {
              if(request->fl_type == F_UNLCK) {
                     if(request->fl_flags & FL_EXISTS)
                            error= -ENOENT;
                     gotoout;
              }
 
              if(!new_fl) {
                     error= -ENOLCK;
                     gotoout;
              }
              locks_copy_lock(new_fl,request);
              locks_move_blocks(new_fl,request);
              locks_insert_lock_ctx(new_fl,&fl->fl_list);
              fl= new_fl;
              new_fl= NULL;
       }
       if(right) {
              if(left == right) {
                     /*The new lock breaks the old one in two pieces,
                      * so we have to use the second new lock.
                      */
                     left= new_fl2;
                     new_fl2= NULL;
                     locks_copy_lock(left,right);
                     locks_insert_lock_ctx(left,&fl->fl_list);
              }
              right->fl_start= request->fl_end + 1;
              locks_wake_up_blocks(right);
       }
       if(left) {
              left->fl_end= request->fl_start - 1;
              locks_wake_up_blocks(left);
       }
 out:
       spin_unlock(&ctx->flc_lock);
       percpu_up_read(&file_rwsem);
       /*
        * Free any unused locks.
        */
       if(new_fl)
              locks_free_lock(new_fl);
       if(new_fl2)
              locks_free_lock(new_fl2);
       locks_dispose_list(&dispose);
       trace_posix_lock_inode(inode,request, error);
 
       returnerror;
}
 
static struct file_lock_context *
locks_get_lock_context(struct inode *inode,int type)
{
       structfile_lock_context *ctx;
 
       /*paired with cmpxchg() below */
       ctx= smp_load_acquire(&inode->i_flctx);
       if(likely(ctx) || type == F_UNLCK)
              gotoout;
       //i_flctx为空，则分配内存
       ctx= kmem_cache_alloc(flctx_cache, GFP_KERNEL);
       if(!ctx)
              gotoout;
 
       spin_lock_init(&ctx->flc_lock);
       INIT_LIST_HEAD(&ctx->flc_flock);
       INIT_LIST_HEAD(&ctx->flc_posix);
       INIT_LIST_HEAD(&ctx->flc_lease);
 
       /*
        * Assign the pointer if it's not alreadyassigned. If it is, then
        * free the context we just allocated.
        */
       if(cmpxchg(&inode->i_flctx, NULL, ctx)) { //inode->i_flctx是NULL，所以将inode->i_flctx指向ctx，并返回NULL
              kmem_cache_free(flctx_cache,ctx);
              ctx= smp_load_acquire(&inode->i_flctx);
       }
out:
       trace_locks_get_lock_context(inode,type, ctx);
       returnctx;
}
 
 
static bool posix_locks_conflict(structfile_lock *caller_fl,
                             struct file_lock *sys_fl)
{
       /*POSIX locks owned by the same process do not conflict with
        * each other.
        */
       if(posix_same_owner(caller_fl, sys_fl)) //锁的拥有者是不是同一个files_struct
              returnfalse;//是同一个files_struct
 
       /*Check whether they overlap */
       if(!locks_overlap(caller_fl, sys_fl)) //校验两个file_lock锁的区域是否有重叠部分
              returnfalse;//没有重叠区域
 
       //caller_fl和sys_fl是不同的files_struct或被锁区域有重叠区域
       returnlocks_conflict(caller_fl, sys_fl);//判断锁的类型，如果是F_WRLCK，则表示有冲突
}
 
static inline int locks_overlap(structfile_lock *fl1, struct file_lock *fl2)
{
       return((fl1->fl_end >= fl2->fl_start) &&
              (fl2->fl_end>= fl1->fl_start));
}
 
/*
 *Check whether two locks have the same owner.
 */
static int posix_same_owner(structfile_lock *fl1, struct file_lock *fl2)
{
       returnfl1->fl_owner == fl2->fl_owner;
}
 
static bool locks_conflict(struct file_lock*caller_fl,
                        struct file_lock *sys_fl)
{
       if(sys_fl->fl_type == F_WRLCK)
              returntrue;
       if(caller_fl->fl_type == F_WRLCK)
              returntrue;
       returnfalse;
}
```
__vfs_write()函数会调用对应文件系统的file_operations，同样，本文以xfs为例进行阐述。
```
static ssize_t __vfs_write(struct file*file, const char __user *p,
                        size_t count, loff_t *pos)
{
       if(file->f_op->write) //如果file有对应的write回调函数
              returnfile->f_op->write(file, p, count, pos);
       elseif (file->f_op->write_iter)
              returnnew_sync_write(file, p, count, pos);
       else
              return-EINVAL;
}
static ssize_t new_sync_write(struct file*filp, const char __user *buf, size_t len, loff_t *ppos)
{
       structiovec iov = { .iov_base = (void __user *)buf, .iov_len = len };
       structkiocb kiocb;
       structiov_iter iter;
       ssize_tret;
 
       init_sync_kiocb(&kiocb,filp);
       kiocb.ki_pos= (ppos ? *ppos : 0);
       iov_iter_init(&iter,WRITE, &iov, 1, len);
 
       ret= call_write_iter(filp, &kiocb, &iter);
       BUG_ON(ret== -EIOCBQUEUED);
       if(ret > 0 && ppos)
              *ppos= kiocb.ki_pos;
       returnret;
}
 
static inline ssize_tcall_write_iter(struct file *file, struct kiocb *kio,
                                  struct iov_iter *iter)
{
       returnfile->f_op->write_iter(kio, iter);
}
```
接下来分析xfs的write_iter()，对应的回调函数是xfs_file_write_iter()。

```
STATIC ssize_t
xfs_file_write_iter(
       structkiocb           *iocb,
       structiov_iter        *from)
{
       structfile        *file = iocb->ki_filp;
       structaddress_space    *mapping =file->f_mapping;
       structinode          *inode = mapping->host;
       structxfs_inode     *ip = XFS_I(inode);
       ssize_t                   ret;
       size_t                    ocount =iov_iter_count(from);
 
       XFS_STATS_INC(ip->i_mount,xs_write_calls);
 
       if(ocount == 0)
              return0;
 
       if(XFS_FORCED_SHUTDOWN(ip->i_mount))
              return-EIO;
 
       if(IS_DAX(inode))//direct write
              returnxfs_file_dax_write(iocb, from);
 
       if(iocb->ki_flags & IOCB_DIRECT) {
              /*
               * Allow a directio write to fall back to abuffered
               * write *only* in the case that we're doing areflink
               * CoW. In all other directio scenarios we do not
               * allow an operation to fall back to bufferedmode.
               */
              ret= xfs_file_dio_aio_write(iocb, from);
              if(ret != -EREMCHG)
                     returnret;
       }
 
       returnxfs_file_buffered_aio_write(iocb, from);//aio write
}
```
主要看下xfs_file_buffered_aio_write()函数，这个函数会对buffer操作，而非direct io(不经过buffer/cache层)

```
xfs_file_buffered_aio_write(
       structkiocb           *iocb,
       structiov_iter        *from)
{
       structfile        *file = iocb->ki_filp;
       structaddress_space    *mapping =file->f_mapping;
       structinode          *inode = mapping->host;
       structxfs_inode     *ip = XFS_I(inode);
       ssize_t                   ret;
       int                 enospc = 0;
       int                 iolock;
 
       if(iocb->ki_flags & IOCB_NOWAIT)
              return-EOPNOTSUPP;
 
write_retry:
       iolock= XFS_IOLOCK_EXCL; //排它锁
       xfs_ilock(ip,iolock);
 
       ret= xfs_file_aio_write_checks(iocb, from, &iolock);
       if(ret)
              gotoout;
 
       /*We can write back this queue in page reclaim */
       current->backing_dev_info= inode_to_bdi(inode);
 
       trace_xfs_file_buffered_write(ip,iov_iter_count(from), iocb->ki_pos);
       ret= iomap_file_buffered_write(iocb, from,
                     &xfs_buffered_write_iomap_ops);//执行write
       if(likely(ret >= 0))
              iocb->ki_pos+= ret;
}
xfs_buffered_write_iomap_ops是个struct:
const struct iomap_ops xfs_buffered_write_iomap_ops= {
       .iomap_begin        = xfs_buffered_write_iomap_begin,
       .iomap_end           = xfs_buffered_write_iomap_end,
};
```

xfs_file_aio_write_checks()函数主要是对待写入文件以及写入数据进行检查

xfs_file_aio_write_checks ()函数调用gereratic_write_checks():
```

inline ssize_t generic_write_checks(structkiocb *iocb, struct iov_iter *from)
{
       structfile *file = iocb->ki_filp;
       structinode *inode = file->f_mapping->host;
       loff_tcount;
       intret;
 
       if(IS_SWAPFILE(inode))
              return-ETXTBSY;
 
       if(!iov_iter_count(from))
              return0;
 
       /*FIXME: this is for backwards compatibility with 2.4 */
       if(iocb->ki_flags & IOCB_APPEND)//追加模式写入
              iocb->ki_pos= i_size_read(inode); //获取当前文件大小，即inode->i_size
 
       if((iocb->ki_flags & IOCB_NOWAIT) && !(iocb->ki_flags &IOCB_DIRECT))
              return-EINVAL;
 
       count= iov_iter_count(from);
       ret= generic_write_check_limits(file, iocb->ki_pos, &count);//判断被写入的字节数是否已经超过当前文件的最大限制
       if(ret)
              returnret;
 
       iov_iter_truncate(from,count);
       returniov_iter_count(from);
}
EXPORT_SYMBOL(generic_write_checks)
 
struct iov_iter {
       /*
        * Bit 0 is the read/write bit, set if we'rewriting.
        * Bit 1 is the BVEC_FLAG_NO_REF bit, set iftype is a bvec and
        * the caller isn't expecting to drop a pagereference when done.
        */
       //描述迭代器类型。它是一个bit位，包含读和写，读写取决于数据是读到迭代器还是从迭代器写入。
       //因此，数据处理方向并不是指迭代器本身，而是数据处理的另外一个部分，即被读操作创建的iov_iter将被写入
       unsignedint type;
       size_tiov_offset;//描述的是由该结构体中iov指针指向的第一个iovec结构体中数据的第一个字节的偏移；
       size_tcount;//iovec数组指向的数据个数存放在count中
       union{
              conststruct iovec *iov;
              conststruct kvec *kvec;
              conststruct bio_vec *bvec;
              structpipe_inode_info *pipe;
       };
       union{
              unsignedlong nr_segs;//iovec结构体的个数存放在nr_segs中
              struct{
                     unsignedint head;
                     unsignedint start_head;
              };
       };
};
```
看下iomap_apply()函数：

```
ssize_t
iomap_file_buffered_write(struct kiocb*iocb, struct iov_iter *iter,
              conststruct iomap_ops *ops)
{
       structinode *inode = iocb->ki_filp->f_mapping->host;
       loff_tpos = iocb->ki_pos, ret = 0, written = 0;
 
       while(iov_iter_count(iter)) {
              ret= iomap_apply(inode, pos, iov_iter_count(iter),
                            IOMAP_WRITE,ops, iter, iomap_write_actor);//基于页来写文件
              if(ret <= 0)
                     break;
              pos+= ret;
              written+= ret;
       }
 
       returnwritten ? written : ret;
}
 
 
loff_t
iomap_apply(struct inode *inode, loff_t pos,loff_t length, unsigned flags,
              conststruct iomap_ops *ops, void *data, iomap_actor_t actor)
{
       structiomap iomap = { .type = IOMAP_HOLE };
       structiomap srcmap = { .type = IOMAP_HOLE };
       loff_twritten = 0, ret;
       u64end;
 
       trace_iomap_apply(inode,pos, length, flags, ops, actor, _RET_IP_);
 
       /*
        * Need to map a range from start position forlength bytes. This can
        * span multiple pages - it is only guaranteedto return a range of a
        * single type of pages (e.g. all into a hole,all mapped or all
        * unwritten). Failure at this point hasnothing to undo.
        *
        * If allocation is required for this range,reserve the space now so
        * that the allocation is guaranteed to succeedlater on. Once we copy
        * the data into the page cache pages, then wecannot fail otherwise we
        * expose transient stale data. If the reservefails, we can safely
        * back out at this point as there is nothingto undo.
        */
       //调用xfs_buffered_write_iomap_begin获取对应xfs的起始位置
       ret= ops->iomap_begin(inode, pos, length, flags, &iomap, &srcmap);
       if(ret)
              returnret;
       if(WARN_ON(iomap.offset > pos))
              return-EIO;
       if(WARN_ON(iomap.length == 0))
              return-EIO;
 
       end= iomap.offset + iomap.length;
       if(srcmap.type != IOMAP_HOLE)
              end= min(end, srcmap.offset + srcmap.length);
       if(pos + length > end)
              length= end - pos;
//通过函数iomap_write_actor（调用iov_iter_copy_from_user_atomic）将数据从用户态复制到内核态。

       written= actor(inode, pos, length, data, &iomap,
                     srcmap.type!= IOMAP_HOLE ? &srcmap : &iomap);
 
       if(ops->iomap_end) {
              ret= ops->iomap_end(inode, pos, length,
                                 written > 0 ? written : 0,
                                 flags, &iomap);//调用xfs_buffered_write_iomap_end
       }
 
       returnwritten ? written : ret;
}
```
iomap_begin、iomap_end分别对应的是xfs_buffered_write_iomap_begin和xfs_buffered_write_iomap_end，主要作用是通过iomap映射将文件块映射到文件系统块，而iomap_write_actor则按照page进行写操作.












