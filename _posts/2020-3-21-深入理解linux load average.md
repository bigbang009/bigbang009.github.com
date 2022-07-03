---
layout:     post
title:      深入理解linux load average
subtitle:   深入理解linux load average
date:       2020-03-21
author:     lwk
catalog: true
tags:
    - linux
    - load average
---
不管是运维还是开发，平时在排查问题系统问题时，经常会使用top命令，top会展示系统当前cpu负载、内存使用等基本信息。其中有一个load average指标，分别表示最近1分钟、5分钟和15分钟的cpu平均负载情况。


![image](https://user-images.githubusercontent.com/36918717/177022349-fec9ae11-64d9-4129-ae3d-28036a872ce0.png)


这个数据是从/proc/loadavg获取的，看下/proc/loadavg里面的内容。


![image](https://user-images.githubusercontent.com/36918717/177022353-95f154bd-bcd8-4915-a3b9-f5b005b7aa12.png)


前三列分别表示最近1分钟、5分钟和15分钟的cpu平均负载（top中的load average就是从这获取的），第四列1/442表示当前正在running的进程数是1，总进程数是442，第五列表示最近一次运行的进程PID号。

那/proc/loadavg前三列cpuload是怎么计算得来的呢？

从内核代码fs/proc/loadavg.c位置看起

![image](https://user-images.githubusercontent.com/36918717/177022356-5dca4f37-870b-401b-9f92-6acee79775d5.png)

loadavg_proc_show函数承担的就是定时更新/proc/loadavg里的cpu负载数据。

其中，LOAD_INT计算avnrun元素的整数部分，LOAG_FRAC计算avnrun元素的小数部分。

负载数据主要通过get_avenrun获得：
![image](https://user-images.githubusercontent.com/36918717/177022360-e3db35b5-1c64-4cd0-8d7c-3abacbd36cb7.png)

#define FSHIFT            11           /* nrof bits of precision */

#define FIXED_1           (1<<FSHIFT)   /*1.0 as fixed-point */

#define LOAD_FREQ    (5*HZ+1)/* 5 sec intervals*/

#define EXP_1              1884              /*1/exp(5sec/1min) as fixed-point */

#define EXP_5              2014              /*1/exp(5sec/5min) */

#define EXP_15            2037              /*1/exp(5sec/15min) */

 

因此FIXED_1/200 = 2048 / 200 = 10，即get_avenrun的参数offset=10，shift=0

数组avenrun大小为3，unsigned long avenrun[3]，存放的就是最近1分钟、5分钟和15分钟的平均负载数据。其中，loadavg是每5秒钟更新一次，即LOAD_FREQ

我们看下avnrun[0]计算过程：

avnrun[0]= (avenrun[0]+FIXED_1/200) <<0=avenrun[0]+(1<<11) /200=avenrun[0]+2048/200

LOAD_INT(avnrun[0])= (avenrun[0]+2048/200)>>11=(avenrun[0]+2048/200)/2048=

avenrun[0]/2048+1/2048

LOAD_FRAC(avnrun[0])= LOAD_INT(((avnrun[0])& (FIXED_1-1)) * 100)=

LOAD_INT(((avnrun[0]) & (2048-1)) *100) = LOAD_INT(((avnrun[0]) & (2048-1)) * 100)=

((avnrun[0]%(2048) * 100) / 2048 + 1/2

 

同理可以推导出LOAD_INT(avnrun[1])、LOAD_INT(avnrun[2])、LOAD_FRAC(avnrun[1])、LOAD_FRAC(avnrun[2])。

那么，只要计算出avenrun，就能计算对应的cpu负载了。

接下来重点看avenrun是如何计算出来的。

avenrun计算逻辑主要在kernel/sched/loadavg.c

![image](https://user-images.githubusercontent.com/36918717/177022371-ec5037c8-e5e0-496d-9094-34abb43f13b2.png)

这里在更新avenrun时，会使用上一次的avenrun的值进行更新。

![image](https://user-images.githubusercontent.com/36918717/177022374-a0051cc5-ba11-41e2-933a-3ce9ea43fa8f.png)
用avenrun(t-1)和avenrun(t)分别表示上一次计算的avenrun和本次计算的avenrun，则根据calc_load可以得到如下计算：

avenrun(t) =(avenrun(t-1)* EXP_N+active*(FIXED_1- EXP_N)) / FIXED_1=

avenrun(t-1)+ ((active *(FIXED_1-EXP_N)) –avenrun(t-1)) * (FIXED_1 -EXP_N)) / FIXED_1

从而推导出：

avenrun(t)- avenrun(t-1)= ((active *(FIXED_1-EXP_N))– avenrun(t-1)) * (FIXED_1 -EXP_N)) / FIXED_1

因此：

(avenrun(t)- avenrun(t-1)) * FIXED_1 =((active *(FIXED_1-EXP_N)) – avenrun(t-1)) * (FIXED_1 -EXP_N))

1分钟、5分钟、15分钟对应的EXP_N值如上，随着EXP_N的增大，(FIXED_1 – EXP_N)/FIXED_1值就越小，这样active的变化对整体load带来的影响就越小。对于一个active波动较小的系统，load会不断的趋近于active，最开始趋近比较快，随着相差值变小，趋近慢慢变缓，越接近时越缓慢，并最终达到active。

那么active又包含哪些，如何计算的？

其实active包含了running和uninterable状态的进程数。

for_each_possible_cpu(cpu){

   nr_active += cpu_of(cpu)->nr_running +cpu_of(cpu)->nr_uninterruptible;

}

avenrun[n] = avenrun[0] * exp_n + nr_active* (1 - exp_n)


![image](https://user-images.githubusercontent.com/36918717/177022380-c8d2b8ea-bb17-49e2-af39-390a55a1b60b.png)

![image](https://user-images.githubusercontent.com/36918717/177022382-0be3f880-d846-4249-84b1-f5a81e044e6a.png)

![image](https://user-images.githubusercontent.com/36918717/177022385-37b0ec91-b75a-4c8a-bbbf-43085aedb34c.png)
![image](https://user-images.githubusercontent.com/36918717/177022387-932a14e2-c089-4edf-8058-7e2dcfa23293.png)

由于2.6之后的内核引入了nohz模式，这就在计算cpu loadavg要稍微负载一些，不仅仅是cpu load，几乎整个涉及到系统性能上的计算都要将nohz模式考虑在内。

Cpu load的nohz模式请参考http://tinylab.org/how-to-calc-load-and-part1/










