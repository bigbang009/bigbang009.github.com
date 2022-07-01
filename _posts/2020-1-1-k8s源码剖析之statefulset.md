---
layout:     post
title:      K8S源码剖析之statefulset
subtitle:   statefulset源码
date:       2020-1-1
author:     lwk
catalog: true
tags:
    - linux
    - k8s
    - statefulset
---
在微服务盛行时代，不深入了解下K8S怎么行！

上一章我们已经通过二进制方法搭建出了一个K8S集群。本章将通过剖析statefulset controller源码，了解statefulset过程。【网上也有对statefulset的分析，不过都是别人的，自己分析一遍比较过瘾】

 

学习之前，先介绍下statefulset，statefulset是K8S其中一种控制器（controller），K8S1.12版本共有20+种controller，所有的controller由kube-controller-manager这个进程启动控制的。

 

 

首先，在我们的测试集群master上看下对应的kube-controller-manager
![image](https://user-images.githubusercontent.com/36918717/176902798-14c33f06-363b-4031-95de-363330ff4c1c.png)
Kube-controller-manager是独立的组件，对应的K8S源码在kubernetes->cmd->kube-controller-manager
![image](https://user-images.githubusercontent.com/36918717/176902844-d6ddf534-5c86-4423-b88e-4343c3ecf663.png)
Statefulset整体的流程图如下所示：

![image](https://user-images.githubusercontent.com/36918717/176902861-d2b035ec-3259-402f-87c3-696e3e1f3e83.png)
找到main函数入口：
![image](https://user-images.githubusercontent.com/36918717/176902882-df430bea-43be-48c1-86fe-2babc68e0161.png)

进入NewControllerManagerCommand函数，该函数调用Run函数，Run函数会负责启动所有的controller

![image](https://user-images.githubusercontent.com/36918717/176902897-03502b9c-cc7f-432d-95f3-d0081eb020a6.png)
NewControllerInitializers主要负责将所有controller加到map中，例如statefulsetcontroller，map存储的key是statefuset，value类型是InitFunc的函数startStatefulSetController

![image](https://user-images.githubusercontent.com/36918717/176902918-913ee418-e676-4897-a0a4-998fcb42ff8a.png)
![image](https://user-images.githubusercontent.com/36918717/176902929-47438f50-fd05-4c78-aa56-033c546091cd.png)
StartControllers遍历mapcontrollers，执行对应的initFn，例如statefulset对应的initFn是startStatefulSetController。

![image](https://user-images.githubusercontent.com/36918717/176902950-c9c4a07b-22a8-4030-86dd-2301757dc138.png)

startStatefulSetController会创建一个goroutine，负责执行statefulset. NewStatefulSetController().Run()。NewStatefulSetController实现对pod和statefulset的add/update/delete事件的注册，并加入到statefulset队列中。

![image](https://user-images.githubusercontent.com/36918717/176902993-ac5a9b1c-1196-47d7-9a84-77f5e8c3cf63.png)
在goroutine中，创建完StatefulSetController之后，执行Run函数，包含两个参数：workers数1和只读的channel变量Stop

![image](https://user-images.githubusercontent.com/36918717/176903033-eb03c0d2-8b27-4de0-8ca0-25774a9efae8.png)

Run函数体里遍历worker数，针对每个worker开启一个goroutine，执行ssc.worker

![image](https://user-images.githubusercontent.com/36918717/176903061-f894d061-98ec-4939-971f-d986e628485e.png)
processNextWorkItem从队列取出statefulset，并调用sync函数进行同步。

![image](https://user-images.githubusercontent.com/36918717/176903089-d5e78382-fe5e-48d5-b032-a25a13287abc.png)

Sync参数是key，类型是string。Key到底是什么呢？长啥一样呢？

![image](https://user-images.githubusercontent.com/36918717/176903110-6daf6dad-3a07-4d34-917b-d14d1a707038.png)

有源码，又搭建了测试集群，那就加点log重新编译出二进制，替换线上版本看看呗。修改statefulset代码，并对kube-controller-manager重新编译（编译指引请参考网上教程）。

![image](https://user-images.githubusercontent.com/36918717/176903132-0a6ddb8a-0e4b-4726-ae40-70ef905a0d69.png)
编译生成新的二进制，替换master的kube-controller-manager并重启kube-controller-manager。

重启后，从/var/log/message可以看到对应的日志输出信息：key值是default/nginx-with-pvc
![image](https://user-images.githubusercontent.com/36918717/176903159-8274aba3-d288-4ee6-b5e2-f455f1fd204c.png)
default/nginx-with-pvc是在测试集群上创建的一个statefulset，默认所在namespace是default


![image](https://user-images.githubusercontent.com/36918717/176903176-f88f928a-1046-4a7f-92ee-202889da8dc0.png)


所以，sync参数的值是namespace_name+statefulset_name。

Sync函数的主要逻辑是：重新获取statefulset对象，获取和该statefulset对象相关的ControllerRevisions；遍历revisions，检查是否有OwnerReference为空的，如果有说明存在Orphaned的revisions，并认领这些Orphanedrevisions；获取和该statefulset对象相关的所有pod；调用ClaimPods检查statefulset和pod的label以及Selector、controllerRef匹配关系。


![image](https://user-images.githubusercontent.com/36918717/176903217-4316f481-02bb-49d3-80d7-ceeac29367f9.png)


![image](https://user-images.githubusercontent.com/36918717/176903226-c875ec2d-e95a-4b2f-8b56-ef07ce3fb5f9.png)


接下来是调用syncStatefulSet，核心逻辑是StatefulSetControlInterface.UpdateStatefulSet

![image](https://user-images.githubusercontent.com/36918717/176903236-e8b72b80-1800-42a8-afd0-c8735bdce7e6.png)

UpdateStatefulSet核心逻辑：获取和该statefulset相关的revisions并进行排序（升序），SortControllerRevisions对revisions的排序是通过byRevision实现sort的接口（Len、Less、Swap）达到对ControllerRevision的排序。

![image](https://user-images.githubusercontent.com/36918717/176903259-766453d6-09e9-4bf7-9ac2-7cbcbf5243c0.png)

getStatefulSetRevisions获取currentRevision和updateRevision。

updateStatefulSet是更新statefulset的核心逻辑：将当前statefulset所拥有的pod进行分组，超出副本集replicaCount的pod被扔进condemned等待销毁；对当前pod数不足情况进行扩容操作；获取副本数组中第一个不健康的Pod；根据副本的序列号检查各个副本的状态（对失败的POD进行重启，）；按照降序删除condemned中的副本；按照降序删除replicas中的副本。

![image](https://user-images.githubusercontent.com/36918717/176903287-4f39ebb6-2ac2-4998-a2d8-00eee2a8f569.png)
![image](https://user-images.githubusercontent.com/36918717/176903301-42835724-e24b-4709-b9e3-00eda51a4311.png)
![image](https://user-images.githubusercontent.com/36918717/176903314-c4f4ef66-a251-48cd-850e-e03db4f6036e.png)
![image](https://user-images.githubusercontent.com/36918717/176903337-f8270500-7fd5-4d54-b0b5-0e71ab411520.png)
![image](https://user-images.githubusercontent.com/36918717/176903345-f3893fc4-29fe-4fd4-ac6f-e45853274e8d.png)

演示效果：创建statefulset，副本数为2
![image](https://user-images.githubusercontent.com/36918717/176903381-5d8b4ffa-720b-45b9-8e58-32f5ef1711ad.png)
扩容：副本数更新到10

kubectl scale statefulset nginx-with-pvc--replicas=10
![image](https://user-images.githubusercontent.com/36918717/176903406-af97e768-a5cb-42bb-b01e-dd79065bd4b0.png)
POD序列号Ordinal从2开始扩容到9。

 

缩容：kubectl patch statefulset nginx-with-pvc  -p'{"spec":{"replicas":2}}'
![image](https://user-images.githubusercontent.com/36918717/176903445-05a73cf5-7f60-4d12-96a3-519deab8d848.png)
按照POD序列号从大到小依次删除。



至此，整个K8S的statefulset流程分析完毕。由于不知专业搞K8S开发的，所以可能有不足的地方，请大家及时指出。



