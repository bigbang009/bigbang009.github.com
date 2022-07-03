---
layout:     post
title:      linux虚拟化之libvirt
subtitle:   linux虚拟化之libvirt
date:       2020-06-22
author:     lwk
catalog: true
tags:
    - linux
    - kernel
    - 虚拟化
    - libvirt
---
本文主要介绍libvirt架构和核心源码剖析。先看下libvirt在整个虚拟化所扮演的角色和位置

![image](https://user-images.githubusercontent.com/36918717/177023975-5945c8ec-d350-4b76-bc65-4b296f26b436.png)
Libvirt主要提供虚拟机管理机制。包括xen/lxc/kvm等。Libvirt包括三个核心部分，API、libvirtd和virsh工具。其中API提供了管理虚拟机的接口，例如创建、启动、重启、销毁等；libvirtd是daemon进程，主要是解析xml文件以及通过RPC调用虚拟化平台（xen/lxc/kvm等）；virsh工具是libvirt提供的控制台工具，可以通过控制台命令执行操作虚拟机相关的指令等。

本文使用Libvirt源码版本4.5.0

首先看下libvirtd的main函数入口，main函数在文件src/remote/remote_daemon.c

![image](https://user-images.githubusercontent.com/36918717/177023979-68fc457b-ff68-4a89-bfac-b8b0fde7c30c.png)

![image](https://user-images.githubusercontent.com/36918717/177023982-750befe1-633e-4447-9561-231bab63bb78.png)
![image](https://user-images.githubusercontent.com/36918717/177023985-fa682b4b-120a-4e49-9c91-9dc0db409ce2.png)


main()函数核心逻辑就是daemonInitialize()，即加载驱动。

![image](https://user-images.githubusercontent.com/36918717/177023989-93da85d9-f096-4886-abcd-c870cad6aa21.png)

这里重点看daemonInitialize()函数对qemu驱动的加载过程。

![image](https://user-images.githubusercontent.com/36918717/177023994-82164262-9385-46bd-b09c-c00613c9b29c.png)

virDriverLoadModule()函数主要是加载驱动，例如qemu，对应的驱动文件是libvirt_driver_qemu.so，通过dlopen打开so文件，然后通过dlsym指向qemuRegister函数地址；紧接着执行(*regsym)()，即执行qemuRegister()。

那么qemuRegister()函数定义在哪？具体做了什么事情？

qemuRegister()定义在src/qemu/qemu_driver.c文件中。（qemu/lxc/vbox/vmware等目录下都有对应的注册函数）

![image](https://user-images.githubusercontent.com/36918717/177023998-7a1ee57f-1b2a-4bb9-a3df-7af65727bf48.png)

![image](https://user-images.githubusercontent.com/36918717/177024001-353256a2-c125-42e7-9a1a-aa857b64d095.png)
当通过libvirtAPI调用virDomainCreate()时，virDomainCreate()定义在src/libvirt-domain.c

![image](https://user-images.githubusercontent.com/36918717/177024003-ea927dca-3d2c-4d41-9333-c110f1f7d56f.png)
domainCreate()调用的是qemuDomainCreate()函数

![image](https://user-images.githubusercontent.com/36918717/177024006-9b1e1b5b-a3d3-45a5-a993-0e4f62bbcf3d.png)
![image](https://user-images.githubusercontent.com/36918717/177024010-fc126e2d-a52d-4ccb-972e-0987593b86ec.png)

qemuDomainCreateWithFlags()函数里核心调用是qemuProcessStart()，即fork一个qemu进程。


![image](https://user-images.githubusercontent.com/36918717/177024014-84ced124-80af-4996-a159-7c782a64a3a6.png)

![image](https://user-images.githubusercontent.com/36918717/177024017-2a620651-e9a8-4d57-a9c8-597e19310558.png)


qemuProcessLaunch()函数主要实现获取qemu驱动配置，初始化日志文件，构建用于创建qemu进程的command、设置最大进程数、最大文件数、最大核心数等。其中virCommandRun()函数实现执行具体command。

![image](https://user-images.githubusercontent.com/36918717/177024021-ee0d9adb-8b55-49d9-9dfe-c4715536743b.png)





virCommandRun()函数调用virCommandRunAsync()，virCommandRunAsync()调用virExec()。

virExec()函数判断qemu可执行二进制文件是否全路径或相对路径，然后fork出一个子进程，在子进程内通过execve/execv执行qemu二进制文件，对应的子进程就是qemu的daemon进程。另外这里说明下，qemu-kvm是老版本的，是qemu的一个分支，从qemu 2.0版本之后，qemu-kvm已经合并到qemu主干，不再有qemu-kvm。从qemu拉下来的最新代码通过—enable-kvm编译出支持KVM版本的qemu。




![image](https://user-images.githubusercontent.com/36918717/177024024-838de532-958f-4086-b4b9-057d1d7cb533.png)

通过fork子进程执行qemu，接下来就进入到qemu逻辑。在上一篇文章qemu逻辑已经大概分析了整个过程。


![image](https://user-images.githubusercontent.com/36918717/177024027-bc711bdc-555b-42b4-b85f-575f5c46059a.png)






