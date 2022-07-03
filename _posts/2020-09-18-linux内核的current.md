---
layout:     post
title:      linux内核的current
subtitle:   linux内核的current
date:       2020-09-18
author:     lwk
catalog: true
tags:
    - linux
    - current
    - kernel
---
Linux内核有个宏叫current，记录当前执行的进程信息。current定义在arch/x86/include/asm/current.h文件中。

![image](https://user-images.githubusercontent.com/36918717/177033470-97cee3c4-d516-4906-bb61-a30c45150b36.png)
从定义可以看出，current对应宏实现是get_current()，而get_current()函数返回的是task_struct结构的指针，即当前执行的进程描述符。

DEFINE_PER_CPU(struct task_struct *, current_task)____cacheline_aligned = &init_task;

看下current_task的定义，默认系统启动时，current_task指向init_task进程。

DEFINE_PER_CPU宏定义在include/linux/percpu-defs.h文件中：

#define DEFINE_PER_CPU(type, name)                                  \

       DEFINE_PER_CPU_SECTION(type,name, "")

 

#define DEFINE_PER_CPU_SECTION(type, name,sec)                          \

       __PCPU_ATTRS(sec)__typeof__(type) name

#endif

 

__PCPU_ATTRS宏定义如下：

#define __PCPU_ATTRS(sec)                                   \

       __percpu__attribute__((section(PER_CPU_BASE_SECTION sec)))     \

       PER_CPU_ATTRIBUTES

PER_CPU_BASE_SECTION宏定义如下：

#define PER_CPU_BASE_SECTION".data..percpu"

 

所以:

DEFINE_PER_CPU(struct task_struct *,current_task) = DEFINE_PER_CPU_SECTION(struct task_struct *, current_task, “”)= __PCPU_ATTRS(“”) __typeof__(struct task_struct *)current_task = __percpu __attribute__((section(“.data..percpu”)))__typeof(struct task_struct *)current_task

 

展开后可以看到实际就是给定义的变量添加section修饰符。和cpu相关的所有变量都会放在对应的section，当访问时在指定的section获取对应的变量。

例如针对进程task_struct，和per-cpu有关系，每个进程在运行时刻只能在一个cpu上执行。

 

Current_task在什么时候会记录当前运行的进程？
![image](https://user-images.githubusercontent.com/36918717/177033484-5eb517fb-deb6-4ba2-aaa2-53672f83347c.png)
__switch_to()函数在文件arch/x86/kernel/process_64.c
![image](https://user-images.githubusercontent.com/36918717/177033508-8b7e76b1-273d-4fa0-9832-2b0693bdd12f.png)

this_cpu_write是宏，

#define this_cpu_write(pcp, val)  __pcpu_size_call(this_cpu_write_, pcp, val)
![image](https://user-images.githubusercontent.com/36918717/177033532-8a75e9c2-0c9e-4d71-ab0b-bb1d0b1cc1fd.png)

this_cpu_write(current_task,next_p) = __pcpu_size_call(this_cpu_write_,current_task,next_p)

所以对应stem是this_cpu_wirte_,variable是current_task

由于sizeof(current_task) = 8（在64位系统）

因此命中case 8: this_cpu_write_8(current_task,__VA_ARGS__)

this_cpu_wirte_8是宏，定义在文件arch/x86/include/asm/percpu.h

#define this_cpu_write_8(pcp, val)      percpu_to_op(volatile, "mov",(pcp), val)

percpu_to_op宏定义如下：

![image](https://user-images.githubusercontent.com/36918717/177033545-11584021-66c7-462b-ae95-699f623d72c4.png)

因此this_cpu_write_8(current_task,__VA_ARGS__)展开：

percpu_to_op(volatile, "mov", (current_task),next_p) =

asm volatile (“movq %1, %%#gs: % #0

: “+m” (current_task)

: “re” （task_struct *）)

关于gs寄存器，这里不做详细阐述了。

 

所以在进行上下文切换的时候会更新current_task变量。

以上就是关于current_task的信息，回到最开始的get_current()函数，函数调用this_cpu_read_stable(current_task)

 

#define this_cpu_read_stable(var)      percpu_stable_op("mov", var)
![image](https://user-images.githubusercontent.com/36918717/177033547-6d8fde47-36f8-48d9-aee5-2305947c28b9.png)
因此，percpu_stable_op("mov", current_task)命中case 8:

asm (“movq  %%gs:%p1,%0”

: “=r” (pfo_ret__)

: “p” (&(current_task))

  总结一下：current是一个宏，当内核要获取当前cpu上执行的进程描述符时，通过get_current读取当前cpu的section里的task_struct。

最后呈现下内核启动流程（后续文章会详细阐述）
![image](https://user-images.githubusercontent.com/36918717/177033556-f2b48117-f9d4-4326-9f3a-e08755bb64c6.png)










