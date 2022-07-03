---
layout:     post
title:      linux系统下free命令内存计算逻辑
subtitle:   linux系统下free命令内存计算逻辑
date:       2021-02-05
author:     lwk
catalog: true
tags:
    - linux
    - free
    - 内存
---

本篇文章介绍下linux下free命令展示的内存相关计算逻辑。本文比较简单（是的，也比较水），主要是做一个抛砖引玉的作用，后续将系统介绍linux内核的内存管理模块。

free命令对应的源码是 https://gitlab.com/procps-ng/procps，其中free.c文件是free命令对应的源码部分。由于现有机器安装的都是3.3.10版本的procps，因此我们下载v3.3.10版本的源码https://gitlab.com/procps-ng/procps/-/tree/v3.3.10

从main函数看起：

```
int main(int argc, char **argv)
{
  int c, flags = 0;
  char *endptr;
  struct commandline_arguments args;

  /*
   * For long options that have no equivalent short option, use a
   * non-character as a pseudo short option, starting with CHAR_MAX + 1.
   */
  enum {
    SI_OPTION = CHAR_MAX + 1,
    TERA_OPTION,
    HELP_OPTION
  };

  static const struct option longopts[] = {
    {  "bytes",  no_argument,      NULL,  'b'    },
    {  "kilo",  no_argument,      NULL,  'k'    },
    {  "mega",  no_argument,      NULL,  'm'    },
    {  "giga",  no_argument,      NULL,  'g'    },
    {  "tera",  no_argument,      NULL,  TERA_OPTION  },
    {  "human",  no_argument,      NULL,  'h'    },
    {  "si",  no_argument,      NULL,  SI_OPTION  },
    {  "lohi",  no_argument,      NULL,  'l'    },
    {  "total",  no_argument,      NULL,  't'    },
    {  "seconds",  required_argument,  NULL,  's'    },
    {  "count",  required_argument,  NULL,  'c'    },
    {  "wide",  no_argument,      NULL,  'w'    },
    {  "help",  no_argument,      NULL,  HELP_OPTION  },
    {  "version",  no_argument,      NULL,  'V'    },
    {  NULL,  0,        NULL,  0    }
  };

  /* defaults */
  args.exponent = 0;
  args.repeat_interval = 1000000;
  args.repeat_counter = 0;

#ifdef HAVE_PROGRAM_INVOCATION_NAME
  program_invocation_name = program_invocation_short_name;
#endif
  setlocale (LC_ALL, "");
  bindtextdomain(PACKAGE, LOCALEDIR);
  textdomain(PACKAGE);
  atexit(close_stdout);

  while ((c = getopt_long(argc, argv, "bkmghltc:ws:V", longopts, NULL)) != -1)
    switch (c) {
    case 'b':
      args.exponent = 1;
      break;
    case 'k':
      args.exponent = 2;
      break;
    case 'm':
      args.exponent = 3;
      break;
    case 'g':
      args.exponent = 4;
      break;
    case TERA_OPTION:
      args.exponent = 5;
      break;
    case 'h':
      flags |= FREE_HUMANREADABLE;
      break;
    case SI_OPTION:
      flags |= FREE_SI;
      break;
    case 'l':
      flags |= FREE_LOHI;
      break;
    case 't':
      flags |= FREE_TOTAL;
      break;
    case 's':
      flags |= FREE_REPEAT;
      args.repeat_interval = (1000000 * strtof(optarg, &endptr));
      if (errno || optarg == endptr || (endptr && *endptr))
        xerrx(EXIT_FAILURE, _("seconds argument `%s' failed"), optarg);
      if (args.repeat_interval < 1)
        xerrx(EXIT_FAILURE,
             _("seconds argument `%s' is not positive number"), optarg);
      break;
    case 'c':
      flags |= FREE_REPEAT;
      flags |= FREE_REPEATCOUNT;
      args.repeat_counter = strtol_or_err(optarg,
        _("failed to parse count argument"));
      if (args.repeat_counter < 1)
        error(EXIT_FAILURE, ERANGE,
          _("failed to parse count argument: '%s'"), optarg);
      break;
    case 'w':
      flags |= FREE_WIDE;
      break;
    case HELP_OPTION:
      usage(stdout);
    case 'V':
      printf(PROCPS_NG_VERSION);
      exit(EXIT_SUCCESS);
    default:
      usage(stderr);
    }

  do {

    meminfo(); //读取/proc/meminfo文件计算对应的内存信息
    /* Translation Hint: You can use 9 character words in
     * the header, and the words need to be right align to
     * beginning of a number. */
    if (flags & FREE_WIDE) {
      printf(_("              total        used        free      shared     buffers       cache   available"));
    } else {
      printf(_("              total        used        free      shared  buff/cache   available"));
    }
    printf("\n");
    printf("%-7s", _("Mem:"));
    printf(" %11s", scale_size(kb_main_total, flags, args));//总内存大小
    printf(" %11s", scale_size(kb_main_used, flags, args));//已经使用内存大小
    printf(" %11s", scale_size(kb_main_free, flags, args));//剩余内存大小
    printf(" %11s", scale_size(kb_main_shared, flags, args));//共享内存大小
    if (flags & FREE_WIDE) {
      printf(" %11s", scale_size(kb_main_buffers, flags, args));
      printf(" %11s", scale_size(kb_main_cached, flags, args));
    } else {
      printf(" %11s", scale_size(kb_main_buffers+kb_main_cached, flags, args));//buffer+cached内存大小
    }
    printf(" %11s", scale_size(kb_main_available, flags, args));
    printf("\n");
    /*
     * Print low vs. high information, if the user requested it.
     * Note we check if low_total == 0: if so, then this kernel
     * does not export the low and high stats. Note we still want
     * to print the high info, even if it is zero.
     */
    if (flags & FREE_LOHI) {//如果有-l选项
      printf("%-7s", _("Low:"));
      printf(" %11s", scale_size(kb_low_total, flags, args));
      printf(" %11s", scale_size(kb_low_total - kb_low_free, flags, args));//used= MemTotal - LowFree
      printf(" %11s", scale_size(kb_low_free, flags, args));
      printf("\n");

      printf("%-7s", _("High:"));
      printf(" %11s", scale_size(kb_high_total, flags, args));
      printf(" %11s", scale_size(kb_high_total - kb_high_free, flags, args));
      printf(" %11s", scale_size(kb_high_free, flags, args));
      printf("\n");
    }

    printf("%-7s", _("Swap:"));
    printf(" %11s", scale_size(kb_swap_total, flags, args));
    printf(" %11s", scale_size(kb_swap_used, flags, args));
    printf(" %11s", scale_size(kb_swap_free, flags, args));
    printf("\n");

    if (flags & FREE_TOTAL) {
      printf("%-7s", _("Total:"));
      printf(" %11s", scale_size(kb_main_total + kb_swap_total, flags, args));
      printf(" %11s", scale_size(kb_main_used + kb_swap_used, flags, args));
      printf(" %11s", scale_size(kb_main_free + kb_swap_free, flags, args));
      printf("\n");
    }
    fflush(stdout);
    if (flags & FREE_REPEATCOUNT) {
      args.repeat_counter--;
      if (args.repeat_counter < 1)
        exit(EXIT_SUCCESS);
    }
    if (flags & FREE_REPEAT) {
      printf("\n");
      usleep(args.repeat_interval);
    }
  } while ((flags & FREE_REPEAT));

  exit(EXIT_SUCCESS);
}

```
重点看下meminfo()函数

```
void meminfo(void){
  char namebuf[32]; /* big enough to hold any row name */
  mem_table_struct findme = { namebuf, NULL};
  mem_table_struct *found;
  char *head;
  char *tail;
  static const mem_table_struct mem_table[] = {
  {"Active",       &kb_active},       // important
  {"Active(file)", &kb_active_file},
  {"AnonPages",    &kb_anon_pages},
  {"Bounce",       &kb_bounce},
  {"Buffers",      &kb_main_buffers}, // important
  {"Cached",       &kb_page_cache},  // important
  {"CommitLimit",  &kb_commit_limit},
  {"Committed_AS", &kb_committed_as},
  {"Dirty",        &kb_dirty},        // kB version of vmstat nr_dirty
  {"HighFree",     &kb_high_free},
  {"HighTotal",    &kb_high_total},
  {"Inact_clean",  &kb_inact_clean},
  {"Inact_dirty",  &kb_inact_dirty},
  {"Inact_laundry",&kb_inact_laundry},
  {"Inact_target", &kb_inact_target},
  {"Inactive",     &kb_inactive},     // important
  {"Inactive(file)",&kb_inactive_file},
  {"LowFree",      &kb_low_free},
  {"LowTotal",     &kb_low_total},
  {"Mapped",       &kb_mapped},       // kB version of vmstat nr_mapped
  {"MemAvailable", &kb_main_available}, // important
  {"MemFree",      &kb_main_free},    // important
  {"MemTotal",     &kb_main_total},   // important
  {"NFS_Unstable", &kb_nfs_unstable},
  {"PageTables",   &kb_pagetables},   // kB version of vmstat nr_page_table_pages
  {"ReverseMaps",  &nr_reversemaps},  // same as vmstat nr_page_table_pages
  {"SReclaimable", &kb_slab_reclaimable}, // "slab reclaimable" (dentry and inode structures)
  {"SUnreclaim",   &kb_slab_unreclaimable},
  {"Shmem",        &kb_main_shared},  // kernel 2.6.32 and later
  {"Slab",         &kb_slab},         // kB version of vmstat nr_slab
  {"SwapCached",   &kb_swap_cached},
  {"SwapFree",     &kb_swap_free},    // important
  {"SwapTotal",    &kb_swap_total},   // important
  {"VmallocChunk", &kb_vmalloc_chunk},
  {"VmallocTotal", &kb_vmalloc_total},
  {"VmallocUsed",  &kb_vmalloc_used},
  {"Writeback",    &kb_writeback},    // kB version of vmstat nr_writeback
  };
  const int mem_table_count = sizeof(mem_table)/sizeof(mem_table_struct);
  unsigned long watermark_low;
  signed long mem_available;

  FILE_TO_BUF(MEMINFO_FILE,meminfo_fd);// MEMINFO_FILE="/proc/meminfo"

  kb_inactive = ~0UL;
  kb_low_total = kb_main_available = 0;

  head = buf;
  for(;;){
    tail = strchr(head, ':');//搜索第一个出现冒号的位置
    if(!tail) break;
    *tail = '\0';
    if(strlen(head) >= sizeof(namebuf)){ //从/proc/meminfo的第一行开始遍历
      head = tail+1;
      goto nextline;
    }
strcpy(namebuf,head);//将head指向的字符串复制到namebuf
//二分查找mem_table中name字符串等于findme->name字符串的，即等于namebuf指向的内容。
    found = bsearch(&findme, mem_table, mem_table_count,
        sizeof(mem_table_struct), compare_mem_table_structs
    );
    head = tail+1;
    if(!found) goto nextline;
    *(found->slot) = (unsigned long)strtoull(head,&tail,10);//这里其实是按照/proc/meminfo中格式和mem_table定义的格式，按照name对应的字符串名，并按照分号分割，将分号后对应的值赋给mem_table每个元素对应的value（个人感觉国外人写代码的想法比较巧妙）。
nextline:
    tail = strchr(head, '\n');
    if(!tail) break;
    head = tail+1;
  }
  if(!kb_low_total){  /* low==main except with large-memory support */
    kb_low_total = kb_main_total;
    kb_low_free  = kb_main_free;
  }
  if(kb_inactive==~0UL){
    kb_inactive = kb_inact_dirty + kb_inact_clean + kb_inact_laundry;
  }
  kb_main_cached = kb_page_cache + kb_slab;
  kb_swap_used = kb_swap_total - kb_swap_free;
  kb_main_used = kb_main_total - kb_main_free - kb_main_cached - kb_main_buffers;

  /* zero? might need fallback for 2.6.27 <= kernel <? 3.14 */
  if (!kb_main_available) {
    if (linux_version_code < LINUX_VERSION(2, 6, 27))
      kb_main_available = kb_main_free;
    else {
      FILE_TO_BUF(VM_MIN_FREE_FILE, vm_min_free_fd);
      kb_min_free = (unsigned long) strtoull(buf,&tail,10);

      watermark_low = kb_min_free * 5 / 4; /* should be equal to sum of all 'low' fields in /proc/zoneinfo */

      mem_available = (signed long)kb_main_free - watermark_low
      + kb_inactive_file + kb_active_file - MIN((kb_inactive_file + kb_active_file) / 2, watermark_low)
      + kb_slab_reclaimable - MIN(kb_slab_reclaimable / 2, watermark_low);

      if (mem_available < 0) mem_available = 0;
      kb_main_available = (unsigned long)mem_available;
    }
  }
}

```
因此
```
total: kb_main_total对应/proc/meminfo的MemTotal，即kb_main_total=MemTotal
free: kb_main_free=MemFree
kb_main_buffers=Buffers
kb_page_cache= Cached
kb_slab= Slab
kb_main_cached=kb_page_cache + kb_slab= Cached+ Slab
used: kb_main_used=kb_main_total - kb_main_free - kb_main_cached - kb_main_buffers
= MemTotal- MemFree- Cached-Slab- Buffers
shared: kb_main_shared= Shmem
buff/cache: kb_main_buffers+kb_main_cached= Buffers+ Cached+ Slab
available: kb_main_available= MemAvailable
```
接下来看下机器实际情况：
```
[root@*******] /data0/src/procps$ cat/proc/meminfo
MemTotal:       16394936 kB
MemFree:        11222348 kB
MemAvailable:   15310428 kB
Buffers:           84676 kB
Cached:          4218800 kB
SwapCached:            0 kB
Active:          3811904 kB
Inactive:        1091868 kB
Active(anon):     604852 kB
Inactive(anon):     1828 kB
Active(file):    3207052 kB
Inactive(file):  1090040 kB
Unevictable:       25048 kB
Mlocked:           25048 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:              1736 kB
Writeback:            64 kB
AnonPages:        620368 kB
Mapped:           605296 kB
Shmem:              2132 kB
KReclaimable:      88120 kB
Slab:             180600 kB
SReclaimable:      88120 kB
SUnreclaim:        92480 kB
KernelStack:        5376 kB
PageTables:        13576 kB
```
 由于free –h会使用1024作为换算单位
```
total= MemTotal=16394936 kB=16394936 / 1024/ 1024= 15G
free= MemFree=11222348 kB = 11222348/1024/1024= 10G
used=MemTotal-MemFree-Cached-Slab-Buffers=(16394936-11222348-4218800-180600-84676)/1024= 672M
shared= Shmem=2132kB = 2132/1024 =2M
buff/cache= Buffers+ Cached+ Slab=(84676kB+4218800kB+180600kB)/1024/1024= 4G
available= MemAvailable=15310428 kB=15310428/1024/1024= 14G
```

free -h显示结果如下：

              total        used        free      shared buff/cache   available

Mem:            15G        672M         10G        2M        4G         14G

Swap:            0B          0B          0B

 

本文主要阐述free命令对于内存部分的计算逻辑，主要是从这篇文章引出下一个篇章的内容，即linux内核的内存管理部分。前面已经对linux的文件系统做了系统性的阐述，接下来的日子会重点投入到内核的内存管理分析方面。






