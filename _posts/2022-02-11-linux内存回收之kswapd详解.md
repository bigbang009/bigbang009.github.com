---
layout:     post
title:      linux内存回收之kswapd详解
subtitle:   linux内存回收之kswapd详解
date:       2022-02-11
author:     lwk
catalog: true
tags:
    - linux
    - kswapd
---

Linux内核内存回收主要有快速内存回收、直接内存回收、kswapd内存回收，这篇文章主要讨论kswapd内存回收，也称为后台回收。

在系统启动过程执行start_kernel的时候会执行kswapd_init，在kswapd_init函数里执行kthread_run，为每个内存节点（node）创建kswapd内核线程。Kswapd内核线程函数体如下所示：

```
static int kswapd(void *p)
{
  unsigned int alloc_order, reclaim_order;
  unsigned int classzone_idx = MAX_NR_ZONES - 1;
  pg_data_t *pgdat = (pg_data_t*)p;
  struct task_struct *tsk = current;
  const struct cpumask *cpumask = cpumask_of_node(pgdat->node_id);

  if (!cpumask_empty(cpumask))
    set_cpus_allowed_ptr(tsk, cpumask);

  /*
   * Tell the memory management that we're a "memory allocator",
   * and that if we need more memory we should get access to it
   * regardless (see "__alloc_pages()"). "kswapd" should
   * never get caught in the normal page freeing logic.
   *
   * (Kswapd normally doesn't need memory anyway, but sometimes
   * you need a small amount of memory in order to be able to
   * page out something else, and this flag essentially protects
   * us from recursively trying to free more memory as we're
   * trying to free the first piece of memory in the first place).
   */
   //标识自己是kswap进程，并允许回写脏页到swap分区
  tsk->flags |= PF_MEMALLOC | PF_SWAPWRITE | PF_KSWAPD;
  set_freezable();

  pgdat->kswapd_order = 0;
  pgdat->kswapd_classzone_idx = MAX_NR_ZONES;
  for ( ; ; ) {
    bool ret;

    alloc_order = reclaim_order = pgdat->kswapd_order;
    classzone_idx = kswapd_classzone_idx(pgdat, classzone_idx);

kswapd_try_sleep:
    //kswap进程尝试睡眠
    kswapd_try_to_sleep(pgdat, alloc_order, reclaim_order,
          classzone_idx);

    /* Read the new order and classzone_idx */
    alloc_order = reclaim_order = pgdat->kswapd_order;
    classzone_idx = kswapd_classzone_idx(pgdat, classzone_idx);
    pgdat->kswapd_order = 0;
    pgdat->kswapd_classzone_idx = MAX_NR_ZONES;

    ret = try_to_freeze();
    if (kthread_should_stop())
      break;

    /*
     * We can speed up thawing tasks if we don't call balance_pgdat
     * after returning from the refrigerator
     */
    if (ret)
      continue;

    /*
     * Reclaim begins at the requested order but if a high-order
     * reclaim fails then kswapd falls back to reclaiming for
     * order-0. If that happens, kswapd will consider sleeping
     * for the order it finished reclaiming at (reclaim_order)
     * but kcompactd is woken to compact for the original
     * request (alloc_order).
     */
    trace_mm_vmscan_kswapd_wake(pgdat->node_id, classzone_idx,
            alloc_order);
    //开始页面回收
    reclaim_order = balance_pgdat(pgdat, alloc_order, classzone_idx);
    if (reclaim_order < alloc_order)
      goto kswapd_try_sleep;
  }

  tsk->flags &= ~(PF_MEMALLOC | PF_SWAPWRITE | PF_KSWAPD);

  return 0;
}
```
看下kswapd_try_to_sleep函数，该函数主要判断kswapd是否满足休眠状态，满足则继续休眠，即调度出去，让出CPU。

```

static void kswapd_try_to_sleep(pg_data_t *pgdat, int alloc_order, int reclaim_order,
        unsigned int classzone_idx)
{
  long remaining = 0;
  DEFINE_WAIT(wait);

  if (freezing(current) || kthread_should_stop())
    return;

  prepare_to_wait(&pgdat->kswapd_wait, &wait, TASK_INTERRUPTIBLE);

  /*
   * Try to sleep for a short interval. Note that kcompactd will only be
   * woken if it is possible to sleep for a short interval. This is
   * deliberate on the assumption that if reclaim cannot keep an
   * eligible zone balanced that it's also unlikely that compaction will
   * succeed.
   */
  if (prepare_kswapd_sleep(pgdat, reclaim_order, classzone_idx)) {
    /*
     * Compaction records what page blocks it recently failed to
     * isolate pages from and skips them in the future scanning.
     * When kswapd is going to sleep, it is reasonable to assume
     * that pages and compaction may succeed so reset the cache.
     */
    reset_isolation_suitable(pgdat);

    /*
     * We have freed the memory, now we should compact it to make
     * allocation of the requested order possible.
     */
    //由于我们目前系统有空闲内存，因此可以尝试内存整理进程
    wakeup_kcompactd(pgdat, alloc_order, classzone_idx);
    //kswap尝试睡眠0.1s，等待唤醒或超时
    remaining = schedule_timeout(HZ/10);

    /*
     * If woken prematurely then reset kswapd_classzone_idx and
     * order. The values will either be from a wakeup request or
     * the previous request that slept prematurely.
     */
       //如果睡眠中途被唤醒，重置kswap相关变量，用于这次的内存回收
    if (remaining) {
      pgdat->kswapd_classzone_idx = kswapd_classzone_idx(pgdat, classzone_idx);
      pgdat->kswapd_order = max(pgdat->kswapd_order, reclaim_order);
    }
    //将kswap从kswapd_wait链表中摘除，并设置为TASK_RUNNING
    finish_wait(&pgdat->kswapd_wait, &wait);
    prepare_to_wait(&pgdat->kswapd_wait, &wait, TASK_INTERRUPTIBLE);
  }

  /*
   * After a short sleep, check if it was a premature sleep. If not, then
   * go fully to sleep until explicitly woken up.
   */
  //如果kswap进程刚才睡眠超时后才返回，也就是remaining=0，意味着kswap可以真正的睡眠
    //也就是可以睡久一点，同时再次确认下kswap进程可以睡眠
  if (!remaining &&
      prepare_kswapd_sleep(pgdat, reclaim_order, classzone_idx)) {
    trace_mm_vmscan_kswapd_sleep(pgdat->node_id);

    /*
     * vmstat counters are not perfectly accurate and the estimated
     * value for counters such as NR_FREE_PAGES can deviate from the
     * true value by nr_online_cpus * threshold. To avoid the zone
     * watermarks being breached while under pressure, we reduce the
     * per-cpu vmstat threshold while kswapd is awake and restore
     * them before going back to sleep.
     */
    set_pgdat_percpu_threshold(pgdat, calculate_normal_threshold);

    if (!kthread_should_stop())
      schedule();//主动调度出去

    set_pgdat_percpu_threshold(pgdat, calculate_pressure_threshold);
  } else {
    if (remaining)
      count_vm_event(KSWAPD_LOW_WMARK_HIT_QUICKLY);
    else
      count_vm_event(KSWAPD_HIGH_WMARK_HIT_QUICKLY);
  }
  finish_wait(&pgdat->kswapd_wait, &wait);
}

```
```

static bool prepare_kswapd_sleep(pg_data_t *pgdat, int order, int classzone_idx)
{
  /*
   * The throttled processes are normally woken up in balance_pgdat() as
   * soon as allow_direct_reclaim() is true. But there is a potential
   * race between when kswapd checks the watermarks and a process gets
   * throttled. There is also a potential race if processes get
   * throttled, kswapd wakes, a large process exits thereby balancing the
   * zones, which causes kswapd to exit balance_pgdat() before reaching
   * the wake up checks. If kswapd is going to sleep, no process should
   * be sleeping on pfmemalloc_wait, so wake them now if necessary. If
   * the wake up is premature, processes will wake kswapd and get
   * throttled again. The difference from wake ups in balance_pgdat() is
   * that here we are under prepare_to_wait().
   */
  if (waitqueue_active(&pgdat->pfmemalloc_wait))
    wake_up_all(&pgdat->pfmemalloc_wait);

  /* Hopeless node, leave it to direct reclaim */
  //kswap回收失败次数达到16次，不再尝试回收，留给进程自行处理
  if (pgdat->kswapd_failures >= MAX_RECLAIM_RETRIES)
    return true;
  //如果至少有一个zone区域满足在high_wmark水位，kswap进程也无需再工作
  if (pgdat_balanced(pgdat, order, classzone_idx)) {
    clear_pgdat_congested(pgdat);
    return true;
  }

  return false;
}
```
```
static bool pgdat_balanced(pg_data_t *pgdat, int order, int classzone_idx)
{
  int i;
  unsigned long mark = -1;
  struct zone *zone;

  /*
   * Check watermarks bottom-up as lower zones are more likely to
   * meet watermarks.
   */
  for (i = 0; i <= classzone_idx; i++) {
    zone = pgdat->node_zones + i;

    if (!managed_zone(zone))//被BUDDY管理的内存为0，则继续
      continue;

    mark = high_wmark_pages(zone);
    if (zone_watermark_ok_safe(zone, order, mark, classzone_idx))
      return true;
  }

  /*
   * If a node has no populated zone within classzone_idx, it does not
   * need balancing by definition. This can happen if a zone-restricted
   * allocation tries to wake a remote kswapd.
   */
  if (mark == -1)
    return true;

  return false;
}
```
```
//函数来更新该zone中与内存规整相关的数据，让本次对zone进行内存规整操作时，
//能够对该zone进行更大范围的页扫描，以此来提高本次内存规整的成功率
void reset_isolation_suitable(pg_data_t *pgdat)
{
  int zoneid;

  for (zoneid = 0; zoneid < MAX_NR_ZONES; zoneid++) {
    struct zone *zone = &pgdat->node_zones[zoneid];
    if (!populated_zone(zone))//除去内存空洞的总物理内存数目
      continue;//为0则继续

    /* Only flush if a full compaction finished recently */
    if (zone->compact_blockskip_flush)//
      __reset_isolation_suitable(zone);//调整内存规整的参数
  }
}
```
看下populated_zone函数
```
/* Returns true if a zone has memory */
static inline bool populated_zone(struct zone *zone)
{
  return zone->present_pages;//present_pages = spanned_pages - absent_pages，absent_pages表示内存空洞的数目
}

//通过__reset_isolation_suitable执行操作可以看出，就是加大内存规整对该zone的页的扫描返回(360度无死角扫描)，
 //力求本次内存规整成功.
static void __reset_isolation_suitable(struct zone *zone)
{
  unsigned long migrate_pfn = zone->zone_start_pfn;
  unsigned long free_pfn = zone_end_pfn(zone) - 1;
  unsigned long reset_migrate = free_pfn;
  unsigned long reset_free = migrate_pfn;
  bool source_set = false;
  bool free_set = false;

  if (!zone->compact_blockskip_flush)
    return;

  zone->compact_blockskip_flush = false;

  /*
   * Walk the zone and update pageblock skip information. Source looks
   * for PageLRU while target looks for PageBuddy. When the scanner
   * is found, both PageBuddy and PageLRU are checked as the pageblock
   * is suitable as both source and target.
   */
  for (; migrate_pfn < free_pfn; migrate_pfn += pageblock_nr_pages,
          free_pfn -= pageblock_nr_pages) {
    cond_resched();

    /* Update the migrate PFN */
    if (__reset_isolation_pfn(zone, migrate_pfn, true, source_set) &&
        migrate_pfn < reset_migrate) {
      source_set = true;
      reset_migrate = migrate_pfn;
      zone->compact_init_migrate_pfn = reset_migrate;
      zone->compact_cached_migrate_pfn[0] = reset_migrate;
      zone->compact_cached_migrate_pfn[1] = reset_migrate;
    }

    /* Update the free PFN */
    if (__reset_isolation_pfn(zone, free_pfn, free_set, true) &&
        free_pfn > reset_free) {
      free_set = true;
      reset_free = free_pfn;
      zone->compact_init_free_pfn = reset_free;
      zone->compact_cached_free_pfn = reset_free;
    }
  }

  /* Leave no distance if no suitable block was reset */
  if (reset_migrate >= reset_free) {
    zone->compact_cached_migrate_pfn[0] = migrate_pfn;
    zone->compact_cached_migrate_pfn[1] = migrate_pfn;
    zone->compact_cached_free_pfn = free_pfn;
  }
}

```
接下来看下balance_pgdat函数，该函数主要实现了内存回收。
```
static int balance_pgdat(pg_data_t *pgdat, int order, int classzone_idx)
{
  int i;
  unsigned long nr_soft_reclaimed;
  unsigned long nr_soft_scanned;
  unsigned long pflags;
  unsigned long nr_boost_reclaim;
  unsigned long zone_boosts[MAX_NR_ZONES] = { 0, };
  bool boosted;
  struct zone *zone;
  struct scan_control sc = {
    .gfp_mask = GFP_KERNEL,
    .order = order,
    .may_unmap = 1,//允许解除页到进程的映射
  };

  set_task_reclaim_state(current, &sc.reclaim_state);
  psi_memstall_enter(&pflags);
  __fs_reclaim_acquire();

  count_vm_event(PAGEOUTRUN);

  /*
   * Account for the reclaim boost. Note that the zone boost is left in
   * place so that parallel allocations that are near the watermark will
   * stall or direct reclaim until kswapd is finished.
   */
  nr_boost_reclaim = 0;
  for (i = 0; i <= classzone_idx; i++) {
    zone = pgdat->node_zones + i;
    if (!managed_zone(zone))//managed_pages为0则continue
      continue;

    nr_boost_reclaim += zone->watermark_boost;//
    zone_boosts[i] = zone->watermark_boost;
  }
  boosted = nr_boost_reclaim;

restart:
  sc.priority = DEF_PRIORITY; //控制每次扫描数量，默认是总页数的1/4096
  do {
    unsigned long nr_reclaimed = sc.nr_reclaimed;
    bool raise_priority = true;
    bool balanced;
    bool ret;

    sc.reclaim_idx = classzone_idx;

    /*
     * If the number of buffer_heads exceeds the maximum allowed
     * then consider reclaiming from all zones. This has a dual
     * purpose -- on 64-bit systems it is expected that
     * buffer_heads are stripped during active rotation. On 32-bit
     * systems, highmem pages can pin lowmem memory and shrinking
     * buffers can relieve lowmem pressure. Reclaim may still not
     * go ahead if all eligible zones for the original allocation
     * request are balanced to avoid excessive reclaim from kswapd.
     */
     //如果buffer_head缓存太多就从最高内存域开始回收
        //buffer_head最多占10%的ZONE_NORMAL
    if (buffer_heads_over_limit) {
      for (i = MAX_NR_ZONES - 1; i >= 0; i--) {
        zone = pgdat->node_zones + i;
        if (!managed_zone(zone))
          continue;

        sc.reclaim_idx = i;
        break;
      }
    }

    /*
     * If the pgdat is imbalanced then ignore boosting and preserve
     * the watermarks for a later time and restart. Note that the
     * zone watermarks will be still reset at the end of balancing
     * on the grounds that the normal reclaim should be enough to
     * re-evaluate if boosting is required when kswapd next wakes.
     */
    //如果有一个zone区域能满足当前内存分区需求，无需回收
        //只有当所有zone都无法满足此order的需求，才会真正去回收内存
        //这也是kswap停止工作的条件
    balanced = pgdat_balanced(pgdat, sc.order, classzone_idx);
    if (!balanced && nr_boost_reclaim) {
      nr_boost_reclaim = 0;
      goto restart;
    }

    /*
     * If boosting is not active then only reclaim if there are no
     * eligible zones. Note that sc.reclaim_idx is not used as
     * buffer_heads_over_limit may have adjusted it.
     */
    if (!nr_boost_reclaim && balanced)
      goto out;

    /* Limit the priority of boosting to avoid reclaim writeback */
    if (nr_boost_reclaim && sc.priority == DEF_PRIORITY - 2)
      raise_priority = false;

    /*
     * Do not writeback or swap pages for boosted reclaim. The
     * intent is to relieve pressure not issue sub-optimal IO
     * from reclaim context. If no pages are reclaimed, the
     * reclaim will be aborted.
     */
    sc.may_writepage = !laptop_mode && !nr_boost_reclaim;
    sc.may_swap = !nr_boost_reclaim;

    /*
     * Do some background aging of the anon list, to give
     * pages a chance to be referenced before reclaiming. All
     * pages are rotated regardless of classzone as this is
     * about consistent aging.
     */
    //如果非活动匿名页太少，对匿名active链表做老化处理，
        //让页面有机会在回收之前被引用，如果系统没有swap分区无需操作
    age_active_anon(pgdat, &sc);

    /*
     * If we're getting trouble reclaiming, start doing writepage
     * even in laptop mode.
     */
     //如果优先级较高，就允许回写操作，以便能回收更多内存
    if (sc.priority < DEF_PRIORITY - 2)
      sc.may_writepage = 1;

    /* Call soft limit reclaim before calling shrink_node. */
    sc.nr_scanned = 0;
    nr_soft_scanned = 0;
    nr_soft_reclaimed = mem_cgroup_soft_limit_reclaim(pgdat, sc.order,
            sc.gfp_mask, &nr_soft_scanned);
    sc.nr_reclaimed += nr_soft_reclaimed;

    /*
     * There should be no need to raise the scanning priority if
     * enough pages are already being scanned that that high
     * watermark would be met at 100% efficiency.
     */
     //针对该节点回收内存页面
    if (kswapd_shrink_node(pgdat, &sc))
      raise_priority = false;

    /*
     * If the low watermark is met there is no need for processes
     * to be throttled on pfmemalloc_wait as they should not be
     * able to safely make forward progress. Wake them
     */
    //kswap处理完回收，判断是否需要让之前等待内存释放而睡眠的进程醒来
        //有可能是回收到了需要的内存，也有可能是kswap无能为力，
    if (waitqueue_active(&pgdat->pfmemalloc_wait) &&
        allow_direct_reclaim(pgdat))
      wake_up_all(&pgdat->pfmemalloc_wait);

    /* Check if kswapd should be suspending */
    __fs_reclaim_release();
    ret = try_to_freeze();
    __fs_reclaim_acquire();
    if (ret || kthread_should_stop())
      break;

    /*
     * Raise priority if scanning rate is too low or there was no
     * progress in reclaiming pages
     */
    nr_reclaimed = sc.nr_reclaimed - nr_reclaimed;
    nr_boost_reclaim -= min(nr_boost_reclaim, nr_reclaimed);

    /*
     * If reclaim made no progress for a boost, stop reclaim as
     * IO cannot be queued and it could be an infinite loop in
     * extreme circumstances.
     */
    if (nr_boost_reclaim && !nr_reclaimed)
      break;
    //如果此次循环没有回收到页面，提高优先级，扫描更多页面
    if (raise_priority || !nr_reclaimed)
      sc.priority--;
  } while (sc.priority >= 1);

  //如果此次kswap没有回收到页面，失败次数加1，达到16次就放弃
  if (!sc.nr_reclaimed)
    pgdat->kswapd_failures++;

out:
  /* If reclaim was boosted, account for the reclaim done in this pass */
  if (boosted) {
    unsigned long flags;

    for (i = 0; i <= classzone_idx; i++) {
      if (!zone_boosts[i])
        continue;

      /* Increments are under the zone lock */
      zone = pgdat->node_zones + i;
      spin_lock_irqsave(&zone->lock, flags);
      zone->watermark_boost -= min(zone->watermark_boost, zone_boosts[i]);
      spin_unlock_irqrestore(&zone->lock, flags);
    }

    /*
     * As there is now likely space, wakeup kcompact to defragment
     * pageblocks.
     */
    wakeup_kcompactd(pgdat, pageblock_order, classzone_idx);
  }

  snapshot_refaults(NULL, pgdat);
  __fs_reclaim_release();
  psi_memstall_leave(&pflags);
  set_task_reclaim_state(current, NULL);

  /*
   * Return the order kswapd stopped reclaiming at as
   * prepare_kswapd_sleep() takes it into account. If another caller
   * entered the allocator slow path while kswapd was awake, order will
   * remain at the higher level.
   */
  return sc.order;
}

```







