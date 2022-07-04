---
layout:     post
title:      drop_caches原理剖析
subtitle:   drop_caches原理剖析
date:       2022-06-18
author:     lwk
catalog: true
tags:
    - linux
    - drop_caches
---

drop_caches是内核对用户态暴露的一个修改sysctl参数的接口，对应的proc文件是/proc/sys/vm/drop_caches，默认值是0，通过修改该值，可以达到回收内存的目的。该值一般有四个选项，对应值分别是1,2,3,4。这篇文章主要阐述drop_caches原理，并且针对回收cache（即vm.drop_caches = 1）进行详解。

drop_caches有效值范围也可以通过GDB kernel的时候查看其min,max值得到，如下图在GDB kernel 5.8版本得到的有效值范围为[1,4]

![image](https://user-images.githubusercontent.com/36918717/177184872-7ef488f9-c750-4cfb-9249-80474b4c286f.png)

针对sysctl内核参数都会有对应的handler处理函数，drop_caches对应的handler是drop_caches_sysctl_handler


```
{    
        .procname   = "drop_caches",
        .data       = &sysctl_drop_caches,
        .maxlen     = sizeof(int),
        .mode       = 0200,
        .proc_handler   = drop_caches_sysctl_handler,
        .extra1     = SYSCTL_ONE,
        .extra2     = &four,
    },
```
重点看下drop_caches_sysctl_handler()函数


```
int drop_caches_sysctl_handler(struct ctl_table *table, int write,
    void __user *buffer, size_t *length, loff_t *ppos)
{
    int ret;

    ret = proc_dointvec_minmax(table, write, buffer, length, ppos);//判断输入是否是有效值
    if (ret)
        return ret;
    if (write) {
        static int stfu;

        if (sysctl_drop_caches & 1) {//如果drop_caches设置的是1
            iterate_supers(drop_pagecache_sb, NULL);//遍历super_block链表，并对每个super_block调用drop_pagecache_sb函数
            count_vm_event(DROP_PAGECACHE);
        }   
        if (sysctl_drop_caches & 2) {//如果drop_caches设置的是2
            drop_slab();
            count_vm_event(DROP_SLAB);
        }   
        if (!stfu) {
            pr_info("%s (%d): drop_caches: %d\n",
                current->comm, task_pid_nr(current),
                sysctl_drop_caches);
        }   
        stfu |= sysctl_drop_caches & 4;
    }   
    return 0;
}
```
iterate_supers()函数会遍历所有super_block文件，即遍历每个挂载点，并遍历挂载点中的每个inode

```
void iterate_supers(void (*f)(struct super_block *, void *), void *arg)
{       
    struct super_block *sb, *p = NULL;
    
    spin_lock(&sb_lock);
    list_for_each_entry(sb, &super_blocks, s_list) {//遍历s_list链表 
        if (hlist_unhashed(&sb->s_instances))
            continue;
        sb->s_count++;
        spin_unlock(&sb_lock);
    
        down_read(&sb->s_umount);
        if (sb->s_root && (sb->s_flags & SB_BORN))
            f(sb, arg);// 并对每个super_block调用drop_pagecache_sb
        up_read(&sb->s_umount);
    
        spin_lock(&sb_lock);
        if (p)
            __put_super(p);
        p = sb;
    }       
    if (p)  
        __put_super(p);
    spin_unlock(&sb_lock);
}
```
通过函数drop_pagecache_sb()函数实现对super_block中pagecache的回收
```
static void drop_pagecache_sb(struct super_block *sb, void *unused)
{
    struct inode *inode, *toput_inode = NULL;
        
    spin_lock(&sb->s_inode_list_lock);
    list_for_each_entry(inode, &sb->s_inodes, i_sb_list) {//遍历super_block的i_sb_list链表，该链表记录的是inode信息
        spin_lock(&inode->i_lock);
        /*
         * We must skip inodes in unusual state. We may also skip
         * inodes without pages but we deliberately won't in case
         * we need to reschedule to avoid softlockups.
         */
        if ((inode->i_state & (I_FREEING|I_WILL_FREE|I_NEW)) ||
            (inode->i_mapping->nrpages == 0 && !need_resched())) {//对inode进行判断，如果inode的状态是I_FREEING|I_WILL_FREE|I_NEW或者inode没有pages并且不需要调度的话就跳过
            spin_unlock(&inode->i_lock);
            continue;
        }
        __iget(inode);
        spin_unlock(&inode->i_lock);
        spin_unlock(&sb->s_inode_list_lock);
        
        cond_resched();
        invalidate_mapping_pages(inode->i_mapping, 0, -1);//对inode的所有为锁定的页面进行遍历，并刷盘之后释放page
        iput(toput_inode);
        toput_inode = inode;
        
        spin_lock(&sb->s_inode_list_lock);
    }   
    spin_unlock(&sb->s_inode_list_lock);
    iput(toput_inode);
}

```
invalidate_mapping_pages()函数遍历时按照15个page的步长进行回收内存，这里不太清楚为啥步长是15。

```
unsigned long invalidate_mapping_pages(struct address_space *mapping,
        pgoff_t start, pgoff_t end)
{
    pgoff_t indices[PAGEVEC_SIZE];
    struct pagevec pvec;
    pgoff_t index = start;
    unsigned long ret;
    unsigned long count = 0;
    int i;
        
    pagevec_init(&pvec);
    while (index <= end && pagevec_lookup_entries(&pvec, mapping, index,
            min(end - index, (pgoff_t)PAGEVEC_SIZE - 1) + 1,
            indices)) { // 每次查找满足条件的最多15个page，并记录在pvec
        for (i = 0; i < pagevec_count(&pvec); i++) {
            struct page *page = pvec.pages[i];
    
            /* We rely upon deletion not changing page->index */
            index = indices[i];
            if (index > end)
                break;
    
            if (xa_is_value(page)) {
                invalidate_exceptional_entry(mapping, index,
                                 page);
                continue;
            }

            if (!trylock_page(page))
                continue;

            WARN_ON(page_to_index(page) != index);

            /* Middle of THP: skip */
            if (PageTransTail(page)) {
                unlock_page(page);
                continue;
            } else if (PageTransHuge(page)) {
                index += HPAGE_PMD_NR - 1;
                i += HPAGE_PMD_NR - 1;
                /*
                 * 'end' is in the middle of THP. Don't
                 * invalidate the page as the part outside of
                 * 'end' could be still useful.
                 */
                if (index > end) {
                    unlock_page(page);
                    continue;
                }

                /* Take a pin outside pagevec */
                get_page(page);

                /*
                 * Drop extra pins before trying to invalidate
                 * the huge page.
                 */
                pagevec_remove_exceptionals(&pvec);
                pagevec_release(&pvec);
            }

            ret = invalidate_inode_page(page);
            unlock_page(page);
            /*
             * Invalidation is a hint that the page is no longer
             * of interest and try to speed up its reclaim.
             */
            if (!ret)
                deactivate_file_page(page);
            if (PageTransHuge(page))
                put_page(page);
            count += ret;
        }
        pagevec_remove_exceptionals(&pvec);
        pagevec_release(&pvec);
        cond_resched();
        index++;
    }
    return count;
}
EXPORT_SYMBOL(invalidate_mapping_pages);
```

```
int invalidate_inode_page(struct page *page)
{               
    struct address_space *mapping = page_mapping(page);
    if (!mapping)
        return 0;
    if (PageDirty(page) || PageWriteback(page))//对于dirty page和writeback page忽略
        return 0;
    if (page_mapped(page))//忽略mapped page，例如页表
        return 0;
    return invalidate_complete_page(mapping, page);
}               
     

static int
invalidate_complete_page(struct address_space *mapping, struct page *page)
{       
    int ret;

    if (page->mapping != mapping)
        return 0;

    if (page_has_private(page) && !try_to_release_page(page, 0))//释放页
        return 0;

    ret = remove_mapping(mapping, page);

    return ret;
}

int try_to_release_page(struct page *page, gfp_t gfp_mask)
{       
    struct address_space * const mapping = page->mapping;
    
    BUG_ON(!PageLocked(page));
    if (PageWriteback(page))
        return 0;

    if (mapping && mapping->a_ops->releasepage)
        return mapping->a_ops->releasepage(page, gfp_mask);//释放页
    return try_to_free_buffers(page);
}

int remove_mapping(struct address_space *mapping, struct page *page)
{   
    if (__remove_mapping(mapping, page, false, NULL)) {
        /*
         * Unfreezing the refcount with 1 rather than 2 effectively
         * drops the pagecache ref for us without requiring another
         * atomic operation.
         */
        page_ref_unfreeze(page, 1);
        return 1;
    }
    return 0;
}   


static int __remove_mapping(struct address_space *mapping, struct page *page,
                bool reclaimed, struct mem_cgroup *target_memcg)
{
    unsigned long flags;
    int refcount;

    BUG_ON(!PageLocked(page));
    BUG_ON(mapping != page_mapping(page));

xa_lock_irqsave(&mapping->i_pages, flags);
    refcount = 1 + compound_nr(page);
    if (!page_ref_freeze(page, refcount))
        goto cannot_free;
    /* note: atomic_cmpxchg in page_ref_freeze provides the smp_rmb */
    if (unlikely(PageDirty(page))) {
        page_ref_unfreeze(page, refcount);
        goto cannot_free;
    }

    if (PageSwapCache(page)) {
        swp_entry_t swap = { .val = page_private(page) };
        mem_cgroup_swapout(page, swap);
        __delete_from_swap_cache(page, swap);
        xa_unlock_irqrestore(&mapping->i_pages, flags);
        put_swap_page(page, swap);
    } else {
        void (*freepage)(struct page *);
        void *shadow = NULL;

        freepage = mapping->a_ops->freepage;
        if (reclaimed && page_is_file_cache(page) &&
            !mapping_exiting(mapping) && !dax_mapping(mapping))
            shadow = workingset_eviction(page, target_memcg);
        __delete_from_page_cache(page, shadow);//删除页
        xa_unlock_irqrestore(&mapping->i_pages, flags);

        if (freepage != NULL)
            freepage(page);
    }

    return 1;

cannot_free:
    xa_unlock_irqrestore(&mapping->i_pages, flags);
    return 0;
}

```

对应的流程图如下所示：

![image](https://user-images.githubusercontent.com/36918717/177185147-4ab21733-18da-4cf6-b1b4-b91bbbc744d4.png)

之所以写这篇文章的原因有两个，一是之前也用到过drop_caches功能，虽然知道是回收内存，但不清楚里面具体的逻辑是什么样的；二是最近遇到过线上机器kswapd导致CPU飙高的问题，通过修改该参数是临时解决kswapd飙高问题的途径之一，所以也是为了加深对drop_caches的理解才深入分析drop_caches的原理。












