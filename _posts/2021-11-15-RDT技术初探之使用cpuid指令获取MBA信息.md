---
layout:     post
title:      RDT技术初探之使用cpuid指令获取MBA信息
subtitle:   RDT技术初探之使用cpuid指令获取MBA信息
date:       2021-11-15
author:     lwk
catalog: true
tags:
    - linux
    - MBA
    - RDT
    - cpuid
---
MBA即Memory Bandwidth Allocation的缩写，即内存带宽分配，是针对LLC隔离的一项技术，也是RDT（Resource Director Technology）的一项指标。

关于RDT技术可参考https://github.com/intel/intel-cmt-cat/wiki/Useful-links

其中，RDT技术需要除了需要硬件支持之外，还需要linux内核支持，linux内核从4.4版本已经支持RDT技术。本文的内核版本基于5.5版本

在内核代码文件arch/x86/kernel/cpu/resctrl/core.c中

```
static bool __get_mem_config_intel(struct rdt_resource *r)
{
    union cpuid_0x10_3_eax eax; 
    union cpuid_0x10_x_edx edx; 
    u32 ebx, ecx; 

    cpuid_count(0x00000010, 3, &eax.full, &ebx, &ecx, &edx.full);//通过cpuid指令获取MBA信息
    r->num_closid = edx.split.cos_max + 1; 
    r->membw.max_delay = eax.split.max_delay + 1; 
    r->default_ctrl = MAX_MBA_BW;
    if (ecx & MBA_IS_LINEAR) {
        r->membw.delay_linear = true;
        r->membw.min_bw = MAX_MBA_BW - r->membw.max_delay;
        r->membw.bw_gran = MAX_MBA_BW - r->membw.max_delay;
    } else {
        if (!rdt_get_mb_table(r))
            return false;
    }    
    r->data_width = 3; 

    r->alloc_capable = true;
    r->alloc_enabled = true;

    return true;
}

/* Some CPUID calls want 'count' to be placed in ecx */
static inline void cpuid_count(unsigned int op, int count,
                   unsigned int *eax, unsigned int *ebx,
                   unsigned int *ecx, unsigned int *edx)
{
    *eax = op;
    *ecx = count;
    __cpuid(eax, ebx, ecx, edx);
}

#define __cpuid         native_cpuid

static inline void native_cpuid(unsigned int *eax, unsigned int *ebx,
                unsigned int *ecx, unsigned int *edx)
{   
    /* ecx is often an input as well as an output. */
    asm volatile("cpuid"
        : "=a" (*eax),
          "=b" (*ebx),
          "=c" (*ecx),
          "=d" (*edx)
        : "0" (*eax), "2" (*ecx)
        : "memory");
}

```

outputs："=a"(a), "=b" (b), "=c" (c), "=d" (d)。表示有4个输出参数——参数0为eax（绑定到变量a）、参数1为ebx（绑定到变量b）、参数2为ecx（绑定到变量c）、参数3为edx（绑定到变量d）。

inputs："0"(*eax), "2" (*ecx))。表示有2个输入参数——参数0（eax），参数2（ecx）。

调用cpuid指令 // asmstatements

将返回的eax赋给a //outputs

将返回的ebx赋给b

将返回的ecx赋给c

将返回的edx赋给d



结合intel的说明文档可以看出，cpuid指令，当eax=16，ecx=3的时候，获取到的是MBA（memory bandwidth allocation）信息

https://www.felixcloutier.com/x86/cpuid
![image](https://user-images.githubusercontent.com/36918717/177043056-831d50c7-e051-47ff-86c4-ef498b5e5e65.png)

这里我将核心代码抠出来，组成可执行程序，看下cpuid执行的结果

```
#include <stdio.h>

typedef unsigned int u32;

union cpuid_0x10_3_eax {
    struct {
        unsigned int max_delay:12;
    } split;
    unsigned int full;
};


union cpuid_0x10_x_edx {
    struct {
        unsigned int cos_max:16;
    } split;
    unsigned int full;
};

static inline void __cpuid(unsigned int *eax, unsigned int *ebx, unsigned int *ecx, unsigned int *edx)
{
    /* ecx is often an input as well as an output. */
    asm volatile("cpuid"
        : "=a" (*eax),
          "=b" (*ebx),
          "=c" (*ecx),
          "=d" (*edx)
        : "0" (*eax), "2" (*ecx)
        : "memory");
}  
   
static inline void cpuid_count(unsigned int op, int count,
                   unsigned int *eax, unsigned int *ebx,
                   unsigned int *ecx, unsigned int *edx)
{
    *eax = op;
    *ecx = count;
    __cpuid(eax, ebx, ecx, edx);
}

int main(int argc, char *argv[]) {

  union cpuid_0x10_3_eax eax;
  union cpuid_0x10_x_edx edx;
  u32 ebx, ecx;

  cpuid_count(0x00000010, 3, &eax.full, &ebx, &ecx, &edx.full);
  printf("%d\n", eax.split.max_delay);

  return 0;
}

```
因此在Intel(R) Xeon(R) Platinum 8260平台上获取到的MBA结果是89

当然上面的程序只能在intel平台上跑起来，如果是AMD平台，则需要修改op，即传给cpuid指定的操作数eax的值。所以针对AMD平台，只需要将cpuid_count(0x00000010,3, &eax.full, &ebx, &ecx, &edx.full);改为cpuid_count(0x80000020,1, &eax.full, &ebx, &ecx, &edx.full);即可。




