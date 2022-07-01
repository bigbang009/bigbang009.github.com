---
layout:     post
title:      linux内核协议栈之TCP四次挥手
subtitle:   linux内核协议栈之TCP四次挥手
date:       2020-01-12
author:     lwk
catalog: true
tags:
    - linux
    - tcp
    - 四次挥手
---

上一篇剖析了TCP三次握手过程，这次看下TCP四次挥手过程，相关内核代码和TCP三次握手有重复地方，所以会粗略截图。

先看下tcp四次挥手抓包情况：
![image](https://user-images.githubusercontent.com/36918717/176907095-676664dc-efd2-4725-986f-31c2b19d1330.png)

从抓包可以看出：

172.***.***.9首先向172.***.***.10发送FIN+ACK包（以下简称9和10）

1、9给10回一个FIN+ACK报文

2、10给9回一个ACK报文

3、10给9又回一个FIN+ACK报文

4、9给10回一个ACK报文

 

9首先调用close，通过系统调用

![image](https://user-images.githubusercontent.com/36918717/176907130-c9f9d855-3d0a-4e5d-9267-28b8761e905a.png)

接着调用__close_fd，获取访问锁，释放文件描述符，释放访问锁，然后调用flip_close

![image](https://user-images.githubusercontent.com/36918717/176907174-77f153c9-b32e-4d25-bc17-2b05fae0a53c.png)

__put_ununsed_fd功能主要是根据待回收的fd确定下一个待分配的fd
![image](https://user-images.githubusercontent.com/36918717/176907195-5d12fc0c-1319-4bb2-8ed7-dfa3eb21f1e4.png)

![image](https://user-images.githubusercontent.com/36918717/176907209-45f14b73-1e9d-4948-bf86-3fc83078bb6a.png)


接下来看下fput的调用
![image](https://user-images.githubusercontent.com/36918717/176907223-1977e822-ec1f-4d92-bf92-449bbb3316d4.png)

![image](https://user-images.githubusercontent.com/36918717/176907235-7cca55cc-593d-4930-ad60-351bafcbac97.png)

![image](https://user-images.githubusercontent.com/36918717/176907248-29dc23d7-79dc-43da-bdc8-ced5a2d503ed.png)

![image](https://user-images.githubusercontent.com/36918717/176907273-adcfdc1b-9aec-4e63-a6ad-5b87bc5dbeb9.png)

![image](https://user-images.githubusercontent.com/36918717/176907307-2279608a-e14b-4207-bcef-bfd6b8ad966f.png)

![image](https://user-images.githubusercontent.com/36918717/176907318-e19910bb-e864-4e69-b7d3-baf1795c57c1.png)

接下来看下sock_close的定义
![image](https://user-images.githubusercontent.com/36918717/176907338-0f04cb85-a067-4ebd-a210-b699abef04d7.png)

![image](https://user-images.githubusercontent.com/36918717/176907356-073c1892-6dc6-4909-ba4e-4488b6449103.png)

其中sock->ops->release最终调用的是net/ipv4/af_inet.cd的 inet_release

![image](https://user-images.githubusercontent.com/36918717/176907397-99ae51ba-0bff-4105-9f8c-8b054f01a63d.png)
![image](https://user-images.githubusercontent.com/36918717/176907409-2e4fa6c4-daa9-4240-8f9b-fb697e02474c.png)

![image](https://user-images.githubusercontent.com/36918717/176907414-9133ac5c-9689-4711-a069-d20c7c20cd85.png)

![image](https://user-images.githubusercontent.com/36918717/176907429-54206b72-1fba-4959-88bf-638eec4f1402.png)

调用tcp_send_fin之后，紧接着就进入了TCP四次挥手的第一次挥手阶段：

![image](https://user-images.githubusercontent.com/36918717/176907456-04962c5c-9f1a-49ba-bd34-7aafaaf25257.png)

以上是第一次挥手过程，即9向10发送fin包。

接下来是第二次挥手：

接着等待另一方回应，从上一篇三次握手可以知道，处理TCP不同状态码的函数为net/ipv4/tcp_input.c中的tcp_rcv_state_process，现在主要是等待对方对fin的ack，让套接字进入fin_wait2状态

![image](https://user-images.githubusercontent.com/36918717/176907483-3493cdb9-4812-40a9-8458-ce3f16657c30.png)

这时，10收到fin报文，同样是tcp_rcv_state_process处理，根据状态（现在是连接建立ESTABLISHED）调用tcp_data_queue进入close_wait状态

![image](https://user-images.githubusercontent.com/36918717/176907509-507c68a2-2224-42eb-82f0-5bf28161fa0c.png)
该函数将套接字状态切换为close_wait，即10状态为close_wait，然后等待新数据发送ack

![image](https://user-images.githubusercontent.com/36918717/176907530-ec8db092-e5a9-4663-9304-607d05cc3557.png)

第三次挥手：

10也调用close，状态由close_wait变为last_ack
![image](https://user-images.githubusercontent.com/36918717/176907562-4c25467c-b9e5-4833-b593-79302de559b0.png)



第四次挥手：

9收到fin报文后，回复ack，并进入time_wait
![image](https://user-images.githubusercontent.com/36918717/176907578-26d233e4-0cb0-4e73-9000-d001f8ed5215.png)
10收到ack后，进入closed状态，回收资源

![image](https://user-images.githubusercontent.com/36918717/176907609-87427feb-2f5a-4c88-8af5-ee30a55eb504.png)
至此，TCP四次挥手完成。

总结一下：
![image](https://user-images.githubusercontent.com/36918717/176907636-1fef9828-495d-4b96-b90e-ecf93aa62ad8.png)









