---
layout:     post
title:      linux内存管理之buddy算法
subtitle:   linux内存管理之buddy算法
date:       2021-05-17
author:     lwk
catalog: true
tags:
    - linux
    - buddy
    - 内存
---

这次主要分析linux内核内存管理的buddy算法，中文名叫伙伴系统。

在之前的文章网卡收包软中断下内核内存分配逻辑以及聊聊kmalloc机制都有阐述内存分配逻辑，即alloc_pages。其中buddy算法就是alloc_pages的一部分：

```

    //rmqueue函数主要实现了伙伴系统算法逻辑 
    page = rmqueue(ac->preferred_zoneref->zone, zone, order,
        gfp_mask, alloc_flags, ac->migratetype);
```
接下来重点分析rmqueue函数


```
/*
 * Allocate a page from the given zone. Use pcplists for order-0 allocations.
 */
 //先考虑从pcp中分配空间，当order大于0时再考虑从伙伴系统中分配
static inline
struct page *rmqueue(struct zone *preferred_zone,
      struct zone *zone, unsigned int order,
      gfp_t gfp_flags, unsigned int alloc_flags,
      int migratetype)
{
  unsigned long flags;
  struct page *page;

  // 如果分配单页, 则进入rmqueue_pcplist
  // 内核中将order=0的请求和大于order=0的请求在处理上做了区分。现在的处理器动不动就十几个核，而zone就那么几个，
  // 当多个核要同时访问同一个zone的时候，不免要在zone的锁的竞争上耗费大量时间。社区开发者发现系统中对order-0的请求在内核中出现的频次极高，
  // order=0所占内存仅一个页的大小，于是就实现了per cpu的"内存池"，用来满足order=0页面的分配，这样就在一定程度上缓解了伙伴系统在zone的锁上面的竞争。
  if (likely(order == 0)) {
    page = rmqueue_pcplist(preferred_zone, zone, gfp_flags,
          migratetype, alloc_flags);
    goto out;
  }

  /*
   * We most definitely don't want callers attempting to
   * allocate greater than order-1 page units with __GFP_NOFAIL.
   */
  WARN_ON_ONCE((gfp_flags & __GFP_NOFAIL) && (order > 1));
  spin_lock_irqsave(&zone->lock, flags);/* 关中断，并获得管理区的锁*/

  do {
    page = NULL;
    if (alloc_flags & ALLOC_HARDER) {
      // 高优先分配：从小到大循环遍历各个order的free_list链表,直到使用get_page_from_free_area()
      // 成功从链表上摘取到最小且合适(order和migratetype都合适)的pageblock
      page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
      if (page)
        trace_mm_page_alloc_zone_locked(page, order, migratetype);
    }
    // 如果未分配到page，则进入__rmqueue
    if (!page)
      page = __rmqueue(zone, order, migratetype, alloc_flags);
  } while (page && check_new_pages(page, order));
  spin_unlock(&zone->lock);
  if (!page)
    goto failed;
  __mod_zone_freepage_state(zone, -(1 << order),
          get_pcppage_migratetype(page));

  __count_zid_vm_events(PGALLOC, page_zonenum(page), 1 << order);
  zone_statistics(preferred_zone, zone);
  local_irq_restore(flags);

out:
  /* Separate test+clear to avoid unnecessary atomics */
  if (test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags)) {
    clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
    wakeup_kswapd(zone, 0, 0, zone_idx(zone));
  }

  VM_BUG_ON_PAGE(page && bad_range(zone, page), page);
  return page;

failed:
  local_irq_restore(flags);
  return NULL;
}

```
rmqueue_pcplist是针对当分配单个page场景下的内存分配逻辑，这个主要是一种内存的缓存机制，因为在大部分场景下分配单page的概率比较大，因此对单page做了一种优化。
```
/* Lock and remove page from the per-cpu list */
static struct page *rmqueue_pcplist(struct zone *preferred_zone,
      struct zone *zone, gfp_t gfp_flags,
      int migratetype, unsigned int alloc_flags)
{
  struct per_cpu_pages *pcp;
  struct list_head *list;
  struct page *page;
  unsigned long flags;
  /*关闭本地中断并保存中断状态（因为中断上下文也可以分配内存）*/
  local_irq_save(flags);
  /*获取当前CPU上目标zone中的per_cpu_pages指针*/
  pcp = &this_cpu_ptr(zone->pageset)->pcp;
  /*获取per_cpu_pages中制定迁移类型的页面list*/
  list = &pcp->lists[migratetype];
  /*从链表上摘取目标页面*/
  page = __rmqueue_pcplist(zone,  migratetype, alloc_flags, pcp, list);
  /*若分配成功，更新当前zone的统计信息*/
  if (page) {
    __count_zid_vm_events(PGALLOC, page_zonenum(page), 1);
    zone_statistics(preferred_zone, zone);
  }
  local_irq_restore(flags);
  return page;
}
```

```

/* Remove page from the per-cpu list, caller must protect the list */
static struct page *__rmqueue_pcplist(struct zone *zone, int migratetype,
      unsigned int alloc_flags,
      struct per_cpu_pages *pcp,
      struct list_head *list)
{
  struct page *page;

  do {
    if (list_empty(list)) {//如果发现页面队列中的页面数为0，需要从伙伴系统中申请一组页面，填充页面队列。
      pcp->count += rmqueue_bulk(zone, 0,
          pcp->batch, list,
          migratetype, alloc_flags);
      if (unlikely(list_empty(list)))
        return NULL;
    }
    // 从队列中取出一页分配出去
    page = list_first_entry(list, struct page, lru);
    list_del(&page->lru);
    pcp->count--;
  } while (check_new_pcp(page));

  return page;
}

```
```
static __always_inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
            int migratetype)
{
  unsigned int current_order;
  struct free_area *area;
  struct page *page;

  /* Find a page of the appropriate size in the preferred list */
  // 遍历order链表，current_order 当前阶，MAX_ORDER最大阶
  for (current_order = order; current_order < MAX_ORDER; ++current_order) {
    area = &(zone->free_area[current_order]);
    page = get_page_from_free_area(area, migratetype);
    if (!page)
      continue;
    del_page_from_free_area(page, area);
    // 调用expand:如果所得到的内存块大于所请求的内存块，则按照伙伴算法的分配原理，将大的页框块分裂成小的页框块
    expand(zone, page, order, current_order, area, migratetype);
    set_pcppage_migratetype(page, migratetype);
    return page;
  }

  return NULL;
}

```
```
//比如当前申请大小为4个page。low表示当前申请页块的阶2，如果阶为2的链表没有空闲块，就一
//直往上找，直到找到阶为4找到空闲块，则high为4。
static inline void expand(struct zone *zone, struct page *page,
  int low, int high, struct free_area *area,
  int migratetype)
{
  unsigned long size = 1 << high;
    //因为只需要4个page，阶为4有16个page，所以剩余有16-4=12个page，这12个page要往阶为2和3的链表上放
    /*由于是从4阶分出的page，因此剩余page合到低阶链表时，肯定是从（4-1）=3阶开始，因此while循环里是high--*/
    //左移1位
    /* 从高阶向低阶迭代 */
  while (high > low) {
    //通过将area指针自减即可得到下级链表
    area--;
    high--;
    size >>= 1; // 右移1位
    VM_BUG_ON_PAGE(bad_range(zone, &page[size]), &page[size]);

    /*
     * Mark as guard pages (or page), that will allow to
     * merge back to allocator when buddy will be freed.
     * Corresponding page table entries will not be touched,
     * pages will stay not present in virtual address space
     */
    if (set_page_guard(zone, &page[size], high, migratetype))
      continue;
    // 后半部分会被留下来，前半部分会继续用于迭代
    add_to_free_area(&page[size], area, migratetype);
    set_page_order(&page[size], high);
  }
}

```
```
static __always_inline struct page *
__rmqueue(struct zone *zone, unsigned int order, int migratetype,
            unsigned int alloc_flags)
{
  struct page *page;

retry:
  // 使用__rmqueue_smallest获得page
  page = __rmqueue_smallest(zone, order, migratetype);
  if (unlikely(!page)) {
    // page分配失败后, 如果迁移类型是MIGRATE_MOVABLE, 进入__rmqueue_cma_fallback
    if (migratetype == MIGRATE_MOVABLE)
      page = __rmqueue_cma_fallback(zone, order);
     // page分配再次失败后使用判断是否可以使用备用迁移类型(如果可以则修改order, migratetype)然后跳转进入retry
    if (!page && __rmqueue_fallback(zone, order, migratetype,
                alloc_flags))
      goto retry;
  }

  trace_mm_page_alloc_zone_locked(page, order, migratetype);
  return page;
}

```
关于内存管理区域zone，和系统的node个数有关系，一般每个node都对应有一个zone，每个zone有几种不同类型的链表。每个链表有11个阶，即从0-10（MAX_ORDER），每一阶表示含有2^order（order表示当前的阶数）个连续page的信息。

下面从具体机器上的/proc/buddyinfo查看内存分布情况：


![image](https://user-images.githubusercontent.com/36918717/177034524-8e198a11-0f71-4f84-92b6-52bb09ce1e9c.png)

用一张图结束本文，下图是buddy数据结构，从0至MAX_ORDER表示连续阶层，例如3：表示连续2^3=8个page的内存链表，

![image](https://user-images.githubusercontent.com/36918717/177034528-5c4d439c-adb1-4981-b892-87875e6b82a0.png)







