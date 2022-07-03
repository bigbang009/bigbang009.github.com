---
layout:     post
title:      RDT技术初探之cbm_mask
subtitle:   RDT技术初探之cbm_mask
date:       2021-11-25
author:     lwk
catalog: true
tags:
    - linux
    - RDT
    - cbm_mask
---

本文主要分析Cache Bit Mask（CBM），在RDT中，cache是通过位掩码描述的，我们使用位掩码描述了可用于分配的缓存部分。掩码的最大值由每个 cpu 模型定义（并且可能针对不同的缓存级别而不同）。它是使用 CPUID 找到的，但也在 resctrl 文件系统的“info/{resource}/cbm_mask”中的“info”目录中提供，例如info/L3/cbm_mask。通过之前的RDT文章我们已经创建出了RDT对应的文件系统出来，在/sys/fs/resctrl/info/L3/目录下有cbm_mask，执行cat命令打印其内容是7ff（注：本文宿主机使用的是Intel(R) Xeon(R) Platinum 8260 CPU）

```
static struct rftype res_common_files[] = {
    {   
       .name       ="cbm_mask",
       .mode       = 0444,
       .kf_ops     =&rdtgroup_kf_single_ops,
       .seq_show   =rdt_default_ctrl_show,
       .fflags     = RF_CTRL_INFO |RFTYPE_RES_CACHE,
}, 
……
……
}
```
同样查看res_common_files结构有对应cbm_mask成员信息。其中seq_show是展示其内容的函数
```

static int rdt_default_ctrl_show(structkernfs_open_file *of,
                 struct seq_file *seq, void *v)
{
   struct rdt_resource *r = of->kn->parent->priv;
 
   seq_printf(seq, "%x\n", r->default_ctrl);
   return 0;
}
```

从rdt_default_ctrl_show()函数可知，cbm_mask内容主要是读取的rdt_resource结构里成员default_ctrl值，那default_ctrl的值是从哪里来的呢？结合之前文章可知，也是在mount的时候通过CPUID获取寄存器的内容得来的。具体函数见get_rdt_alloc_resources()
```

 
static __initbool get_rdt_alloc_resources(void)
{
    bool ret = false;
 
    if (rdt_alloc_capable)
        return true;
 
    if (!boot_cpu_has(X86_FEATURE_RDT_A))
        return false;
 
    if (rdt_cpu_has(X86_FEATURE_CAT_L3)) {
        rdt_get_cache_alloc_cfg(1,&rdt_resources_all[RDT_RESOURCE_L3]);
        if (rdt_cpu_has(X86_FEATURE_CDP_L3))
            rdt_get_cdp_l3_config();
        ret = true;
    }
    if (rdt_cpu_has(X86_FEATURE_CAT_L2)) {
        /* CPUID 0x10.2 fields are same formatat 0x10.1 */
        rdt_get_cache_alloc_cfg(2,&rdt_resources_all[RDT_RESOURCE_L2]);
        if (rdt_cpu_has(X86_FEATURE_CDP_L2))
            rdt_get_cdp_l2_config();
        ret = true;
    }
 
    if (get_mem_config())
        ret = true;
 
    return ret;
}
```

其中rdt_get_cache_alloc_cfg()通过CPUID获取num_closid、cbm_len以及default_ctrl等信息。

```
static voidrdt_get_cache_alloc_cfg(int idx, struct rdt_resource *r)
{      
    union cpuid_0x10_1_eax eax;
    union cpuid_0x10_x_edx edx;
    u32 ebx, ecx;
       
    cpuid_count(0x00000010, idx, &eax.full,&ebx, &ecx, &edx.full);
    r->num_closid = edx.split.cos_max + 1;
    r->cache.cbm_len = eax.split.cbm_len +1;
    r->default_ctrl =BIT_MASK(eax.split.cbm_len + 1) - 1;
    r->cache.shareable_bits = ebx &r->default_ctrl;
    r->data_width = (r->cache.cbm_len +3) / 4;
    r->alloc_capable = true;
    r->alloc_enabled = true;
}  
```
同样，我将核心代码抠出，并构造出可编译执行的程序。

```
#include <stdio.h>
 
#ifdef__ASSEMBLY__
#define_AC(X,Y)    X
#define_AT(T,X)    X
#else
#define__AC(X,Y)   (X##Y)
#define_AC(X,Y)    __AC(X,Y)
#define_AT(T,X)    ((T)(X))
#endif
 
#define_UL(x)      (_AC(x, UL))
#define_ULL(x)     (_AC(x, ULL))
 
#define _BITUL(x)   (_UL(1) << (x))
#define_BITULL(x)  (_ULL(1) << (x))
 
 
#defineUL(x)       (_UL(x))
#defineULL(x)      (_ULL(x))
 
 
typedef unsignedint u32;
 
#ifdefCONFIG_64BIT
#defineBITS_PER_LONG 64
#else
#defineBITS_PER_LONG 32
#endif /*CONFIG_64BIT */
 
 
#defineBIT_MASK(nr)        (UL(1) << ((nr)% BITS_PER_LONG))
 
/*CPUID.(EAX=10H, ECX=ResID=1).EAX */
unioncpuid_0x10_1_eax {
    struct {
        unsigned int cbm_len:5;
    } split;
    unsigned int full;
};
/*CPUID.(EAX=10H, ECX=ResID).EDX */
union cpuid_0x10_x_edx{
    struct {
        unsigned int cos_max:16;
    } split;
    unsigned int full;
};
 
static inlinevoid __cpuid(unsigned int *eax, unsigned int *ebx, unsigned int *ecx, unsignedint *edx)
{
    /* ecx is often an input as well as anoutput. */
    asm volatile("cpuid"
        : "=a" (*eax),
          "=b" (*ebx),
          "=c" (*ecx),
          "=d" (*edx)
        : "0" (*eax), "2"(*ecx)
        : "memory");
} 
  
static inlinevoid cpuid_count(unsigned int op, int count,
                   unsigned int *eax, unsignedint *ebx,
                   unsigned int *ecx, unsignedint *edx)
{
    *eax = op;
    *ecx = count;
    __cpuid(eax, ebx, ecx, edx);
}
 
 
int main(intargc, char *argv[]) {
 
       union cpuid_0x10_1_eax eax;
       union cpuid_0x10_x_edx edx;
       u32 ebx, ecx;
       cpuid_count(0x00000010, 1, &eax.full,&ebx, &ecx, &edx.full);
       printf("num_closid=%d\n",edx.split.cos_max + 1);
       printf("cbm_len=%d\n",eax.split.cbm_len + 1); // cache.cbm_len
       printf("default_ctrl=%d\n",BIT_MASK(eax.split.cbm_len + 1) - 1);
       printf("shareable_bits=%d\n",ebx & (BIT_MASK(eax.split.cbm_len + 1) - 1)); // cache.shareable_bits
       printf("data_width=%d\n",(eax.split.cbm_len + 1 + 3) / 4);
 
 
       return 0;
}
```
编译后执行结果如下：

num_closid=16

cbm_len=11

default_ctrl=2047

shareable_bits=1536

data_width=3

从运行结果看default_ctrl的值是2047，对应16进制就是0x7ff，和实际机器上的info/L3/cbm_mask的内容一致

![image](https://user-images.githubusercontent.com/36918717/177043541-306e046e-5eb6-4afe-80be-5ac783bec5f1.png)


以上内容就是在Intel(R) Xeon(R) Platinum 8260 CPU机器上通过CPUID指令获取cbm_mask的信息，RDT技术通过提供kernfs从而提供对外接口，涉及到的cbm_mask都是通过CPUID指令从寄存器中获取到的值。






