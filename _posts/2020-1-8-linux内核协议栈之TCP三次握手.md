---
layout:     post
title:      linux内核协议栈之TCP三次握手
subtitle:   linux内核协议栈之TCP三次握手
date:       2020-01-08
author:     lwk
catalog: true
tags:
    - linux
    - tcp
    - kernel
---

搞IT开发和运维的基本上都知道TCP三次握手和四次挥手，但你知道TCP三次握手里每一次握手内核都做了什么事情吗？本文将带你从内核协议栈了解TCP三次握手的逻辑。

看这篇文章之前还是要先对linux系统调用有一定的了解，可参考之前的文章说一说linux的系统调用



 

客户端调用connect通过系统调用__x64_sys_connect

arch/x86/entry/syscalls/syscall_64.tbl

![image](https://user-images.githubusercontent.com/36918717/176906022-be255d5d-9167-4a71-852f-517e0ae0c870.png)

#define SYS_CONNECT     3               /* sys_connect(2)               */

![image](https://user-images.githubusercontent.com/36918717/176906042-da135de6-4db6-47b5-8723-6ec16c8ffa9b.png)

![image](https://user-images.githubusercontent.com/36918717/176906056-6639c54b-2f7f-4539-8e7a-db169dc4e469.png)

由于本文阐述的是流式套接字，即TCP，因此sock->ops->connect最终调用inet_stream_connect

inet_stream_connect定义在net/ipv4/af_inet.c
![image](https://user-images.githubusercontent.com/36918717/176906081-7688e33b-e019-4a2d-838f-d5d8b41537be.png)
![image](https://user-images.githubusercontent.com/36918717/176906092-ade02804-3753-430c-8668-e74653bbefc9.png)
__inet_stream_connect主要功能是检查地址长度和协议族；对于TCP，调用tcp_v4_connect；对于阻塞调用，等待后续握手的完成；对于非阻塞调用，则直接返回 -EINPROGRESS

![image](https://user-images.githubusercontent.com/36918717/176906113-9f5e6751-eee9-4202-a3a4-7c97df3c14b2.png)
因为sk->sk_prot是structproto 类型，struct proto声明了connect接口，因此sk->sk_prot->connect调用的是tcp_v4_connect ，所在文件是net/ipv4/tcp_ipv4.c

![image](https://user-images.githubusercontent.com/36918717/176906159-156c3f75-cf8b-4ba1-a2de-888a8b13b441.png)

tcp_v4_connect核心逻辑如下：

![image](https://user-images.githubusercontent.com/36918717/176906175-8ed837cd-5811-4608-af07-1516ab701e7a.png)

其中ip_route_newinet_hash_connect是内核分配端口号逻辑，在上一篇linux内核端口分配策略已经阐述，这里不再赘述。

这里核心函数是tcp_connect，主要功能是根据 sk 中的信息，申请sk_buff空间，将skb初始化为syn报文，调用tcp_connect_queue_skb将报文添加到发送队列sk->sk_write_queue中，并调用tcp_transmit_skb构造TCP头，然后交给网络层处理，最后初始化重传定时器

![image](https://user-images.githubusercontent.com/36918717/176906203-d625f737-d027-4d99-979f-af37e8a7fe90.png)

以上是TCP三次握手的第一次握手，即client向server发送syn报文过程。

 

接下来看下TCP三次握手的第二次握手过程，即server向client发送sync+ack报文

第一次握手，client向server发送syn报文后，正常情况下server端收到syn报文，给client回syn+ack报文。整体逻辑是：网卡->netif_receive_skb()--->ip_rcv()--->ip_local_deliver_finish()--->tcp_v4_rcv()

（PS：从网卡接收到包以及内核对包分配内存等逻辑在后续的文章会介绍）

tcp_v4_rcv核心逻辑如下：

![image](https://user-images.githubusercontent.com/36918717/176906228-d601e562-db96-4940-931a-504fa8ab1855.png)
![image](https://user-images.githubusercontent.com/36918717/176906238-14601ccb-aaf6-4fe6-a415-a4baa86323b3.png)

![image](https://user-images.githubusercontent.com/36918717/176906258-2738988e-d9f9-4f44-8d8e-bc05093ff4b4.png)
![image](https://user-images.githubusercontent.com/36918717/176906271-25c3ed60-fcda-4c06-b403-be694d0f9886.png)
![image](https://user-images.githubusercontent.com/36918717/176906279-6617a270-066c-4ae9-8c82-37a5c02cd9b7.png)

tcp_conn_request代码比较多，就不贴了，这里主要说下tcp_conn_request的核心逻辑：

1、分配一个request_sock对象来代表这次连接请求（状态为TCP_NEW_SYN_RECV），如果没有设置防范syn flood相关的选项，则将该request_sock添加到established状态的tcp_sock散列表（如果设置了防范选项，则request_sock对象都没有，只有建立完成时才会分配）

2、调用af_ops->send_synack即tcp_v4_send_synack发送syn+ack，进行第二次握手

 ![image](https://user-images.githubusercontent.com/36918717/176906304-fc7e40b9-b353-4641-aff5-6e6d4b70be65.png)

tcp_v4_send_synack主要逻辑：

1、查找客户端路由，即server给client发送syn+ack时，需要知道下一条路由

2、生成syn+ack报文，并产生ip数据报依靠网络层将数据报发送出去

3、客户端socket状态变为TCP_ESTABLISHED，此时服务端socket的状态为TCP_NEW_SYN_RECV

 接下来是第三次握手

客户端收到服务端返回的syn+ack报文之后，给服务端回复ack报文，

![image](https://user-images.githubusercontent.com/36918717/176906352-f76f7e00-48ab-4fb0-9b42-816dddd3fd55.png)

其中tcp_check_req：

1 通过调用链tcp_v4_syn_recv_sock--> tcp_create_openreq_child --> inet_csk_clone_lock 生成新sock，状态设置为TCP_SYN_RECV；且tcp_v4_syn_recv_sock通过调用inet_ehash_nolisten将新sock加入ESTABLISHED状态的哈希表中;


![image](https://user-images.githubusercontent.com/36918717/176906376-bd097e32-8224-46e1-84a0-be95d3d6b838.png)
2 通过调用inet_csk_complete_hashdance，将新sock插入accept队列.

 

至此我们得到一个代表本次连接的新sock，状态为TCP_SYN_RECV，接着调用tcp_child_process，进而调用tcp_rcv_state_process：

和第二次握手类似，又回到了tcp_rcv_state_process
![image](https://user-images.githubusercontent.com/36918717/176906411-e17b33e1-db89-400b-bf7b-0f1c02bafd1a.png)

Client端设置状态为ESTABLISHED，至此TCP三次握手完成。

总结一下：

1、首先客户端创建sock对象，构造syn报文，并以此创建ip数据报，通过网络层将ip数据报发送给服务端。

2、服务端收到ack报文，将request_sock插入队列，并构造syn+ack报文，通过网络层传给客户端

3、客户端收到syn+ack报文后，状态转为TCP_ESTABLISHED，并构造ack报文，发送给服务端

 ![image](https://user-images.githubusercontent.com/36918717/176906443-400be0e1-8c9d-4fd6-9cd1-e5913ce83f7a.png)

 

