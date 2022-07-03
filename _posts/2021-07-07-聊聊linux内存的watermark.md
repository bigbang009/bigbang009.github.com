---
layout:     post
title:      聊聊linux内存的watermark
subtitle:   聊聊linux内存的watermark
date:       2021-07-07
author:     lwk
catalog: true
tags:
    - linux
    - watermark
    - 内存
    - min_free_kbytes
---

linux内存的watermark又称为水位，是linux内存管理的一个知识点，水位决定了内存的使用情况，以及系统是否进行内存回收以及如何回收的策略问题。

内存水位源码位于mm/memory_hotplug.c的init_per_zone_wmark_min()函数以及mm/page_alloc.c的min_free_kbytes_sysctl_handler()函数中。其中min_free_kbytes_sysctl_handler()是通过proc虚拟文件系统修改min_free_kbytes参数达到修改内存水位的值，而init_per_zone_wmark_min()函数是内存初始化的时候设置的内存水位值，两者的差别在于init_per_zone_wmark_min()设置的水位值在[128KB,64MB]之间，而min_free_kbytes_sysctl_handler()函数可以设置为任意值。

下面从源码分析内存水位

```

int __meminit init_per_zone_wmark_min(void)
{
    unsigned long lowmem_kbytes;
    int new_min_free_kbytes;

    lowmem_kbytes = nr_free_buffer_pages() * (PAGE_SIZE >> 10); 
    new_min_free_kbytes = int_sqrt(lowmem_kbytes * 16); 

    if (new_min_free_kbytes > user_min_free_kbytes) {
        min_free_kbytes = new_min_free_kbytes;
        if (min_free_kbytes < 128) 
            min_free_kbytes = 128; 
        if (min_free_kbytes > 65536)
            min_free_kbytes = 65536;
    } else {
        pr_warn("min_free_kbytes is not updated to %d because user defined value %d is preferred\n",
                new_min_free_kbytes, user_min_free_kbytes);
    }    
    setup_per_zone_wmarks();
    refresh_zone_stat_thresholds();
    setup_per_zone_lowmem_reserve();

#ifdef CONFIG_NUMA
    setup_min_unmapped_ratio();
    setup_min_slab_ratio();
#endif

    return 0;
```
```

// ZONE_DMA和ZONE_NORMAL页面数之和
unsigned long nr_free_buffer_pages(void)
{
    return nr_free_zone_pages(gfp_zone(GFP_USER));
}

#define ___GFP_DIRECT_RECLAIM   0x400u
#define ___GFP_KSWAPD_RECLAIM   0x800u
#define ___GFP_IO       0x40u
#define ___GFP_FS       0x80u
#define ___GFP_HARDWALL     0x100000u

GFP_USER = (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
 = ((__force gfp_t)(___GFP_DIRECT_RECLAIM|___GFP_KSWAPD_RECLAIM)) | ((__force gfp_t)___GFP_IO) | ((__force gfp_t)___GFP_FS) | ((__force gfp_t)___GFP_HARDWALL)
= （0x400u | 0x800u） | （0x40u） | （0x80u）|（0x100000u）
=1051840

```
```
static inline enum zone_type gfp_zone(gfp_t flags)
{
    enum zone_type z;
    int bit = (__force int) (flags & GFP_ZONEMASK); // bit=1051840 & 15 = 0

    z = (GFP_ZONE_TABLE >> (bit * GFP_ZONES_SHIFT)) &
                     ((1 << GFP_ZONES_SHIFT) - 1); // 18743602 & 3 = 2
    VM_BUG_ON((GFP_ZONE_BAD >> bit) & 1);
    return z; // 2
}
```
```
根据上面计算出的GFP_USER可知
参数flags是(0x400u | 0x800u) | (0x40u) | (0x80u)|(0x100000u)
```
```

#define ___GFP_DMA      0x01u
#define ___GFP_HIGHMEM      0x02u
#define ___GFP_DMA32        0x04u
#define ___GFP_MOVABLE      0x08u
GFP_ZONEMASK = (__GFP_DMA|__GFP_HIGHMEM|__GFP_DMA32|__GFP_MOVABLE)
= ((__force gfp_t)___GFP_DMA) | ((__force gfp_t)___GFP_HIGHMEM) | ((__force gfp_t)___GFP_DMA32) | ((__force gfp_t)___GFP_MOVABLE)
= 15
```
```

#define GFP_ZONES_SHIFT 2

#define GFP_ZONE_TABLE ( \
    (ZONE_NORMAL << 0 * GFP_ZONES_SHIFT)                       \
    | (OPT_ZONE_DMA << ___GFP_DMA * GFP_ZONES_SHIFT)               \
    | (OPT_ZONE_HIGHMEM << ___GFP_HIGHMEM * GFP_ZONES_SHIFT)           \
    | (OPT_ZONE_DMA32 << ___GFP_DMA32 * GFP_ZONES_SHIFT)               \
    | (ZONE_NORMAL << ___GFP_MOVABLE * GFP_ZONES_SHIFT)            \
    | (OPT_ZONE_DMA << (___GFP_MOVABLE | ___GFP_DMA) * GFP_ZONES_SHIFT)    \
    | (ZONE_MOVABLE << (___GFP_MOVABLE | ___GFP_HIGHMEM) * GFP_ZONES_SHIFT)\
    | (OPT_ZONE_DMA32 << (___GFP_MOVABLE | ___GFP_DMA32) * GFP_ZONES_SHIFT)\
)
= (2 << 0 * 2) | (0 << 0x01u * 2) | (3 << 0x02u * 2) | (1 << 0x04u * 2) | (2 << 0x08u * 2) | (3 << (0x08u | 0x01u) * 2) | (4 << (0x08u | 0x01u) * 2) | (1 << (0x08u | 0x04u) * 2)
= 18743602
```
```
int __meminit init_per_zone_wmark_min(void)
{
    unsigned long lowmem_kbytes;
    int new_min_free_kbytes;
  // 获取系统空闲内存大小(扣除high水位)
    lowmem_kbytes = nr_free_buffer_pages() * (PAGE_SIZE >> 10); 
    new_min_free_kbytes = int_sqrt(lowmem_kbytes * 16); 

    if (new_min_free_kbytes > user_min_free_kbytes) {
        min_free_kbytes = new_min_free_kbytes;
        if (min_free_kbytes < 128) 
            min_free_kbytes = 128; //最小是128KB 
        if (min_free_kbytes > 65536)
            min_free_kbytes = 65536;// /最大65M，但是这只是系统初始化的值，可以通过proc接口设置范围外的值
    } else {
        pr_warn("min_free_kbytes is not updated to %d because user defined value %d is preferred\n",
                new_min_free_kbytes, user_min_free_kbytes);
    }    
  //设置每个zone的min low high水位值
    setup_per_zone_wmarks();
    refresh_zone_stat_thresholds();
  //设置每个zone为其他zone的保留内存
    setup_per_zone_lowmem_reserve();

#ifdef CONFIG_NUMA
    setup_min_unmapped_ratio();
    setup_min_slab_ratio();
#endif

    return 0;
}



void setup_per_zone_wmarks(void)
{
  static DEFINE_SPINLOCK(lock);

  spin_lock(&lock);
  __setup_per_zone_wmarks();
  spin_unlock(&lock);
}


struct zone {
  /* Read-mostly fields */

  /* zone watermarks, access with *_wmark_pages(zone) macros */
  unsigned long _watermark[NR_WMARK];
  unsigned long watermark_boost;

  unsigned long nr_reserved_highatomic;

  /*
   * We don't know if the memory that we're going to allocate will be
   * freeable or/and it will be released eventually, so to avoid totally
   * wasting several GB of ram we must reserve some of the lower zone
   * memory (otherwise we risk to run OOM on the lower zones despite
   * there being tons of freeable ram on the higher zones).  This array is
   * recalculated at runtime if the sysctl_lowmem_reserve_ratio sysctl
   * changes.
   */
  long lowmem_reserve[MAX_NR_ZONES];

#ifdef CONFIG_NUMA
  int node;
#endif
  struct pglist_data  *zone_pgdat;
  struct per_cpu_pageset __percpu *pageset;

#ifndef CONFIG_SPARSEMEM
  /*
   * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
   * In SPARSEMEM, this map is stored in struct mem_section
   */
  unsigned long    *pageblock_flags;
#endif /* CONFIG_SPARSEMEM */

  /* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
  unsigned long    zone_start_pfn;

  /*
   * spanned_pages is the total pages spanned by the zone, including
   * holes, which is calculated as:
   *   spanned_pages = zone_end_pfn - zone_start_pfn;
   *
   * present_pages is physical pages existing within the zone, which
   * is calculated as:
   *  present_pages = spanned_pages - absent_pages(pages in holes);
   *
   * managed_pages is present pages managed by the buddy system, which
   * is calculated as (reserved_pages includes pages allocated by the
   * bootmem allocator):
   *  managed_pages = present_pages - reserved_pages;
   *
   * So present_pages may be used by memory hotplug or memory power
   * management logic to figure out unmanaged pages by checking
   * (present_pages - managed_pages). And managed_pages should be used
   * by page allocator and vm scanner to calculate all kinds of watermarks
   * and thresholds.
   *
   * Locking rules:
   *
   * zone_start_pfn and spanned_pages are protected by span_seqlock.
   * It is a seqlock because it has to be read outside of zone->lock,
   * and it is done in the main allocator path.  But, it is written
   * quite infrequently.
   *
   * The span_seq lock is declared along with zone->lock because it is
   * frequently read in proximity to zone->lock.  It's good to
   * give them a chance of being in the same cacheline.
   *
   * Write access to present_pages at runtime should be protected by
   * mem_hotplug_begin/end(). Any reader who can't tolerant drift of
   * present_pages should get_online_mems() to get a stable value.
   */
  atomic_long_t    managed_pages;
  unsigned long    spanned_pages;
  unsigned long    present_pages;

  const char    *name;

#ifdef CONFIG_MEMORY_ISOLATION
  /*
   * Number of isolated pageblock. It is used to solve incorrect
   * freepage counting problem due to racy retrieving migratetype
   * of pageblock. Protected by zone->lock.
   */
  unsigned long    nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
  /* see spanned/present_pages for more description */
  seqlock_t    span_seqlock;
#endif

  int initialized;

  /* Write-intensive fields used from the page allocator */
  ZONE_PADDING(_pad1_)

  /* free areas of different sizes */
  struct free_area  free_area[MAX_ORDER];

  /* zone flags, see below */
  unsigned long    flags;

  /* Primarily protects free_area */
  spinlock_t    lock;

  /* Write-intensive fields used by compaction and vmstats. */
  ZONE_PADDING(_pad2_)

  /*
   * When free pages are below this point, additional steps are taken
   * when reading the number of free pages to avoid per-cpu counter
   * drift allowing watermarks to be breached
   */
  unsigned long percpu_drift_mark;

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
  /* pfn where compaction free scanner should start */
  unsigned long    compact_cached_free_pfn;
  /* pfn where async and sync compaction migration scanner should start */
  unsigned long    compact_cached_migrate_pfn[2];
  unsigned long    compact_init_migrate_pfn;
  unsigned long    compact_init_free_pfn;
#endif

#ifdef CONFIG_COMPACTION
  /*
   * On compaction failure, 1<<compact_defer_shift compactions
   * are skipped before trying again. The number attempted since
   * last failure is tracked with compact_considered.
   */
  unsigned int    compact_considered;
  unsigned int    compact_defer_shift;
  int      compact_order_failed;
#endif

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
  /* Set to true when the PG_migrate_skip bits should be cleared */
  bool      compact_blockskip_flush;
#endif

  bool      contiguous;

  ZONE_PADDING(_pad3_)
  /* Zone statistics */
  atomic_long_t    vm_stat[NR_VM_ZONE_STAT_ITEMS];
  atomic_long_t    vm_numa_stat[NR_VM_NUMA_STAT_ITEMS];
} ____cacheline_internodealigned_



static void __setup_per_zone_wmarks(void)
{
  unsigned long pages_min = min_free_kbytes >> (PAGE_SHIFT - 10);//最小空闲内存页数
  unsigned long lowmem_pages = 0;
  struct zone *zone;
  unsigned long flags;

  /* Calculate total number of !ZONE_HIGHMEM pages */
  for_each_zone(zone) { //遍历每个zone
    if (!is_highmem(zone))
      lowmem_pages += zone_managed_pages(zone); //代表的是除过HIGHMEM_ZONE的所有zone的managed_pages
  }

  for_each_zone(zone) {
    u64 tmp;

    spin_lock_irqsave(&zone->lock, flags);
    tmp = (u64)pages_min * zone_managed_pages(zone);//用每个zone的manage_pagest的页数乘以pages_min的值
    do_div(tmp, lowmem_pages);//得到的值就是min的值
    if (is_highmem(zone)) {
      /*
       * __GFP_HIGH and PF_MEMALLOC allocations usually don't
       * need highmem pages, so cap pages_min to a small
       * value here.
       *
       * The WMARK_HIGH-WMARK_LOW and (WMARK_LOW-WMARK_MIN)
       * deltas control async page reclaim, and so should
       * not be capped for highmem.
       */
      unsigned long min_pages;

      min_pages = zone_managed_pages(zone) / 1024;
      min_pages = clamp(min_pages, SWAP_CLUSTER_MAX, 128UL);
      zone->_watermark[WMARK_MIN] = min_pages;
    } else {
      /*
       * If it's a lowmem zone, reserve a number of pages
       * proportionate to the zone's size.
       */
      zone->_watermark[WMARK_MIN] = tmp;//设置MIN水位的值
    }

    /*
     * Set the kswapd watermarks distance according to the
     * scale factor in proportion to available memory, but
     * ensure a minimum size on small systems.
     */
    tmp = max_t(u64, tmp >> 2,
          mult_frac(zone_managed_pages(zone),
              watermark_scale_factor, 10000));

    zone->_watermark[WMARK_LOW]  = min_wmark_pages(zone) + tmp;//LOW水位的值=MIN+MIN/4=125%MIN
    zone->_watermark[WMARK_HIGH] = min_wmark_pages(zone) + tmp * 2; //HIGH水位的值=MIN + MIN/2 = 150%MIN
    zone->watermark_boost = 0;

    spin_unlock_irqrestore(&zone->lock, flags);
  }

  /* update totalreserve_pages */
  calculate_totalreserve_pages();
}


/*
 * setup_per_zone_lowmem_reserve - called whenever
 *  sysctl_lowmem_reserve_ratio changes.  Ensures that each zone
 *  has a correct pages reserved value, so an adequate number of
 *  pages are left in the zone after a successful __alloc_pages().
 */
static void setup_per_zone_lowmem_reserve(void)
{
  struct pglist_data *pgdat;
  enum zone_type j, idx;

  for_each_online_pgdat(pgdat) {
    //遍历每个zone，假设系统上有ZONE_DMA，ZONE_DMA32，ZONE_NORMAL三个zone类型
    for (j = 0; j < MAX_NR_ZONES; j++) {
      struct zone *zone = pgdat->node_zones + j;
      unsigned long managed_pages = zone_managed_pages(zone);

      zone->lowmem_reserve[j] = 0;

      idx = j;
      while (idx) {
        struct zone *lower_zone;

        idx--;
        lower_zone = pgdat->node_zones + idx;

        if (sysctl_lowmem_reserve_ratio[idx] < 1) {
          sysctl_lowmem_reserve_ratio[idx] = 0;
          //j=0,则zone[DMA].lowmem_reserve[DMA] = 0
                //j=1,则zone[DMA32].lowmem_reserve[DMA32] = 0
                  //j=2,则zone[NORMAL].lowmem_reserve[NORMAL] = 0
                  //这里的意思就是对于自身zone的内存使用，不需要考虑保留
          lower_zone->lowmem_reserve[j] = 0;
        } else {
          //j=0,不会进入循环
                  //j=1,idx=0,则zone[DMA].lowmem_reserve[DMA32] = zone[DMA32].pages/sysctl_lowmem_reserve_ratio[DMA]
                  //j=2,idx=1,则zone[DMA32].lowmem_reserve[NORMAL] = zone[NORMAL].pages/sysctl_lowmem_reserve_ratio[DMA32]
                  //    idx=0,则zone[DMA].lowmem_reserve[NORMAL] = zone[NORMAL+DMA32].pages/sysctl_lowmem_reserve_ratio[DMA] 
          lower_zone->lowmem_reserve[j] =
            managed_pages / sysctl_lowmem_reserve_ratio[idx];
        }
        managed_pages += zone_managed_pages(lower_zone);
      }
    }
  }

  /* update totalreserve_pages */
  calculate_totalreserve_pages();
}

设置lowmem_reserve后就会再次更新totalreserve_pages的值
/*
 * calculate_totalreserve_pages - called when sysctl_lowmem_reserve_ratio
 *  or min_free_kbytes changes.
 */
static void calculate_totalreserve_pages(void)
{
  struct pglist_data *pgdat;
  unsigned long reserve_pages = 0;
  enum zone_type i, j;
  //遍历每个node
  for_each_online_pgdat(pgdat) {

    pgdat->totalreserve_pages = 0;
    //遍历每个zone
    for (i = 0; i < MAX_NR_ZONES; i++) {
      struct zone *zone = pgdat->node_zones + i;
      long max = 0;
      unsigned long managed_pages = zone_managed_pages(zone);

      /* Find valid and maximum lowmem_reserve in the zone */
      //查找当前zone中，为上级zone内存类型最大的保留值
      for (j = i; j < MAX_NR_ZONES; j++) {
        if (zone->lowmem_reserve[j] > max)
          max = zone->lowmem_reserve[j];
      }

      /* we treat the high watermark as reserved pages. */
      //每个zone的high水位值和最大保留值之和当做是系统运行保留阈值
      max += high_wmark_pages(zone);

      if (max > managed_pages)
        max = managed_pages;

      pgdat->totalreserve_pages += max;

      reserve_pages += max;
    }
  }
  //这个totalreserve_pages在overcommit时会使用到
    //该值作为系统正常运行的最低保证
  totalreserve_pages = reserve_pages;
}

```
找台机器验证一把：
```
[root@dev-1]/home/ansible$ cat /proc/zoneinfo | grep -E "Node|managed|min"
Node 0,zone      DMA
        min     16
        managed 3977
Node 0,zone    DMA32
        min     3011
        managed 724692
Node 0,zone   Normal
        min     13867
        managed 3337777
```
```
[root@dev-1]/home/ansible$ cat /proc/sys/vm/min_free_kbytes
67584
Node[0].DMA.min=3977/4*67584/(3977+724692+3337777)=16
Node[0].DMA32.min=724692/4*67584/(3977+724692+3337777)=3011

```
要注意cat/proc/zoneinfo | grep -E "Node|managed|min"显示的数字单位是页数，即pages，所以Node[0].DMA.min就是对应zone的内存总页数乘以系统保留内存占总内存的百分比

