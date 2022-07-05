---
layout:     post
title:      一次centos8上ethtool显示问题
subtitle:   一次centos8上ethtool显示问题
date:       2021-09-05
author:     lwk
catalog: true
tags:
    - linux
    - ethtool
---

在新机房发现虚拟机的宿主机获取网卡基础信息时显示的是无效数据，如下图所示：

![image](https://user-images.githubusercontent.com/36918717/177042628-4d59796b-c3a9-4768-8735-a9c6c52ba87e.png)

新机房虚拟机的宿主机使用的镜像版本是CentOS Linux release 8.4.2105，kernel是5.10.26-2，经IDC团队排查，和内核版本无关系，但和ethtool版本有关系，centos8的默认带的是5.8版本，使用低版本的ethtool就没问题。

我下载了ethtool的源码包，分别有5.7/5.8/5.9版本，将编译后的二进制放在新机房的虚拟机测试机发现，5.8/5.9版本会出现展示无效数字问题，而低于5.8版本的就没这个问题。如下图所示：

![image](https://user-images.githubusercontent.com/36918717/177042637-214b3d61-1906-4d1c-9030-79104963256a.png)
![image](https://user-images.githubusercontent.com/36918717/177042639-2b8d9f4f-4563-41d3-a549-20c3e8089e53.png)

那么ethtool从5.7版本到5.8版本做了什么修改才会导致展示的数字无效呢？

 我们先把5.8编译出二进制，并GDB调试下
 
 ![image](https://user-images.githubusercontent.com/36918717/177042651-fb90e591-632e-43a3-92d9-cbec2a0694e9.png)

接下来我们编译5.7版本的二进制，并且GDB看下：

![image](https://user-images.githubusercontent.com/36918717/177042657-21d0c4fc-2ff7-40f5-9a51-0c8986c17a9d.png)

我们把args打印出来看到结果如下：
```
{{opts = 0x42a275 "-s|--change",no_dev = false, func = 0x408390 <do_sset>, nlfunc = 0x427750<nl_sset>, help = 0x42b60a "Change generic options"}} 
...
{opts = 0x42b92c"-l|--show-channels", no_dev = false, func = 0x403df0<do_gchannels>,
   nlfunc = 0x0, help = 0x42b93f "Query Channels", xhelp = 0x0}
...
  
```
 从这里可以看到，args[27]的数据是
 ```
 {opts = 0x42b92c "-l|--show-channels", no_dev = false,func = 0x403df0 <do_gchannels>,
    nlfunc = 0x0, help = 0x42b93f "QueryChannels", xhelp = 0x0}
 ```

可以看出args[27].nlfunc=0x0。

 

但为什么args[27].nlfunc没有对应函数呢？

![image](https://user-images.githubusercontent.com/36918717/177042678-fb4c689c-c0f6-4d3e-badc-64ff359f946b.png)

因此，低于5.8版本会正常的进入到do_gchannels()函数，最终通过ioctl系统调用获取网卡配置信息
![image](https://user-images.githubusercontent.com/36918717/177042688-ea000e86-4935-4bbe-9307-8c6787d21fd4.png)


版本号是5.7以及版本号是5.8的在centos7上会走到ioctl系统调用获取网卡队列信息；版本号是5.7的在centos8上也是走到ioctl系统调用获取网卡队列信息；版本号是5.8在centos8上通过netlink获取网卡队列信息，在调用show_u32()函数转换时，由于attr是空，所以print n/a.

至于attr为什么为空，主要是由于通过netlink获取到的值为0导致。





