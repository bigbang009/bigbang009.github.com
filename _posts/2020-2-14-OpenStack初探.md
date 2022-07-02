---
layout:     post
title:      OpenStack初探
subtitle:   OpenStack初探
date:       2020-02-14
author:     lwk
catalog: true
tags:
    - linux
    - OpenStack
    - iaas
---
这几天无心研究内核，因此就去看了看OpenStack相关内容，和之前看K8S类似，先从整体架构去看，然后拆分每个组件的去看。毕竟一个大项目短期内不可能吃透，所以先有整体把握，再逐个深入。好了，废话少说，先说下什么是OpenStack。

OpenStack是一个由NASA（美国国家航空航天局）和Rackspace合作研发并发起的，以Apache许可证授权的自由软件和开放源代码项目。

OpenStack 是一个开源的云计算管理平台项目，由几个主要的组件组合起来完成具体工作。OpenStack支持几乎所有类型的云环境，项目目标是提供实施简单、可大规模扩展、丰富、标准统一的云计算管理平台。OpenStack通过各种互补的服务提供了基础设施即服务（IaaS）的解决方案，每个服务提供API以进行集成。

OpenStack是一个旨在为公共及私有云的建设与管理提供软件的开源项目。它的社区拥有超过 130家企业及1350位开发者，这些机构与个人都将OpenStack作为基础设施即服务（IaaS）资源的通用前端。OpenStack项目的首要任务是简化云的部署过程并为其带来良好的可扩展性。本文希望通过提供必要的指导信息，帮助大家利用OpenStack前端来设置及管理自己的共有云或私有云。

OpenStack云计算平台，帮助服务商和企业内部实现类似于 Amazon EC2 和 S3 的云基础架构服务(Infrastructure as a Service, IaaS)。OpenStack核心模块主要包括：nova（计算服务）、neutron（网络服务）、swift（对象存储）、cinder（块存储）、glance（镜像管理）、keystone（认证服务）、horizon（UI）等。

OpenStack既是一个社区，也是一个项目和一个开源软件，它提供了一个部署云的操作平台或工具集。其宗旨在于，帮助组织运行为虚拟计算或存储服务的云，为公有云、私有云，也为大云、小云提供可扩展的、灵活的云计算。

首先我们看下OpenStack逻辑架构
![image](https://user-images.githubusercontent.com/36918717/177003293-a1cadfe3-3996-4bd9-ae79-36235c09fdf6.png)
OpenStack裸机服务-Ironic

   简而言之，OpenStack Ironic就是一个进行裸机部署安装的项目。

    所谓裸机，就是指没有配置操作系统的计算机。从裸机到应用还需要进行以下操作：

  （1）硬盘RAID、分区和格式化；

  （2）安装操作系统、驱动程序；

  （3）安装应用程序。

    Ironic实现的功能，就是可以很方便的对指定的一台或多台裸机，执行以上一系列的操作。例如部署大数据群集需要同时部署多台物理机，就可以使用Ironic来实现。

    Ironic可以实现硬件基础设施资源的快速交付。

 

 

Ironic架构：
![image](https://user-images.githubusercontent.com/36918717/177003305-1040b8e7-27a2-405c-bda7-cbf3d2d9f0c6.png)

Ironic服务由以下组件组成:

（1）基于RESTful的API服务。操作者和其它服务通过API与裸金属管理服务器交互。

（2）Conductor服务。Ironic的核心组件，通过API提供功能调用。Conductor服务和API服务通过RPC通信。

（3）各种驱动程序，支持不同的硬件。

（4）消息队列RabbitMQ。

 

（5）数据库。用于存储信息资源，除此之外,还包括存储Conductor、节点(物理服务器)和drivers等信息。

    用户通过Nova API和Nova Scheduler来启动一个裸金属实例，之后请求会通过Ironic API，连接到Ironic Conductor服务，再到对应的Driver，最后完成实例部署，为用户提供成功部署的物理机服务。

 

OpenStack Telemetry-计量&告警服务

计量数据收集（Telemetry）服务提供如下功能：

相关OpenStack服务的有效调查计量数据。

通过监测通知收集来自各个服务发送的事件和计量数据。

发布收集来的数据到多个目标，包括数据存储和消息队列。

Telemetry架构：


![image](https://user-images.githubusercontent.com/36918717/177003313-38aa1132-ce8e-4940-a75d-08e4967db014.png)
主要有四个部分组成：

Gnocchi：时间序列数据库，保存计量数据，Gnocchi接收来自Ceilometer的原始计量数据，进行聚合运算后保存到持久化后端。

Panko：事件数据库，保存事件数据，Panko接收来自Ceilometer的事件数据，并提供查询接口。

Ceilometer：数据采集服务，采集资源使用量相关数据，并推送到Gnocchi；采集操作事件数据，并推送到Panko。

Aodh：告警服务，基于计量和事件数据提供告警通知功能。

 

其实，telemetry可以看作是由计费+监控告警量大部分组成。目前公有云厂商的计费模式基本上是按照服务的API调用次数进行统计计费的。另外就是监控告警，这里可以实现对具体客户、服务以及接口的监控和告警（告警通道可以通过短信、邮件、电话等形式）。

Ceilometer服务包含以下组件

计算代理 (ceilometer-agent-compute)

运行在每个计算节点中，推送资源的使用状态，也许在未来会有其他类型的代理，但是目前来说社区专注于创建计算节点代理。

中心代理 (ceilometer-agent-central)

运行在中心管理服务器以推送资源使用状态，既不捆绑到实例也不在计算节点。代理可启动多个以横向扩展它的服务。

ceilometer通知代理；

运行在中心管理服务器(s)中，获取来自消息队列(s)的消息去构建事件和计量数据。

ceilometor收集器（负责接收信息进行持久化存储）

运行在中心管理服务器(s),分发收集的telemetry数据到数据存储或者外部的消费者，但不会做任何的改动。

API服务器 (ceilometer-api)

运行在一个或多个中心管理服务器，提供从数据存储的数据访问。

 

检查告警服务

当收集的度量或事件数据打破了界定的规则时，计量报警服务会出发报警。

计量报警服务包含以下组件：

API服务器 (aodh-api)

运行于一个或多个中心管理服务器上提供访问存储在数据中心的警告信息。

报警评估器 (aodh-evaluator)

运行在一个或多个中心管理服务器，当警告发生是由于相关联的统计趋势超过阈值以上的滑动时间窗口，然后作出决定。

通知监听器 (aodh-listener)

运行在一个中心管理服务器上，来检测什么时候发出告警。根据对一些事件预先定义一些规则，会产生相应的告警，同时能够被Telemetry数据收集服务的通知代理捕获到。

报警通知器 (aodh-notifier)

运行在一个或多个中心管理服务器，允许警告为一组收集的实例基于评估阀值来设置。

 

Gnocchi-时序数据库服务

gnocchi-api：提供数据传入接口，接收原始计量数据，并将它们保存到传入数据后端。同时提供聚合计量数据的查询接口，从聚合数据后端读取计量数据返回给用户。

gnocchi-metricd：从传入数据后端读取原始计量数据，进行聚合计算，然后将聚合数据保存到聚合数据后端。

Panko-事件存储服务

panko-api: 提供事件数据的插入和查询接口

这些服务使用OpenStack消息总线来通信，只有收集者和API服务可以访问数据存储。

OpenStack Orchestration service

Orchestration中文译为编排，官网解释为基于模板的编排服务，其中heat是OpenStack的核心编排服务。通过运行open-stack API调用生成正在运行的云应用程序来描述云应用程序。该软件将OpenStack的其他核心组件集成到一个文件模板系统中。模板允许您创建大多数OpenStack资源类型，例如实例，浮动IP，卷，安全组和用户。用户预先定义一个规定格式的任务模板（模板是yaml格式，类似K8S的资源和配置定义格式），任务模板中定义了一连串的相关任务（例如用某配置开几台虚拟机，然后在其中一台安装一个mysql服务，设定相关的数据库属性，然后再配置几台虚拟机安装web服务集群等等），然后将模板交由heat执行，heat就会按一定得顺序执行heat模板中定义的一连串任务

它还提供了高级功能，例如实例高可用性，实例自动缩放和嵌套堆栈。这使OpenStack核心项目能够获得更大的用户群。

该服务使部署人员可以直接或通过自定义插件与Orchestration服务集成。

编排服务架构：

![image](https://user-images.githubusercontent.com/36918717/177003329-41aa2597-40db-4038-99fb-cea7472d59e6.png)
编排服务主要由以下部分构成：

Heat:命令行客户端，与heat-api通信以运行AWS CloudFormation API，最终开发人员可以直接使用Orchestration REST API

Heat-api：OpenStack本地API，通过远程调用(RPC)将请求发送到heat-engine来处理API请求

Heat-api-cfn：与AWS CloudFormation兼容的AWS Query API，通过RPC将API请求发送到heat-engine来处理它们

Heat-engine：协调模板启动并将事件返回至API使用者

                                                      

当 Heat Engine 拿到请求后，会把请求解析为各种类型的资源，每种资源都对应 OpenStack 其它的服务客户端，然后通过发送 REST 的请求给其它服务。通过如此的解析和协作，最终完成请求的处理。

 

Heat Engine 在这里的作用分为三层：第一层处理 Heat 层面的请求，就是根据模板和输入参数来创建 Stack，这里的 Stack 是由各种资源组合而成。第二层解析 Stack 里各种资源的依赖关系，Stack 和嵌套 Stack 的关系。第三层就是根据解析出来的关系，依次调用各种服务客户段来创建各种资源。

 

 

OpenStack 对象存储-Swift服务

OpenStack Swift 开源项目提供了弹性可伸缩、高可用的分布式对象存储服务，适合存储大规模非结构化数据。

Swift架构：
![image](https://user-images.githubusercontent.com/36918717/177003339-674495f9-3919-471a-91c6-7d165ce091bd.png)
Swift 组件包括：

 

代理服务（Proxy Server）：对外提供对象服务 API，会根据环的信息来查找服务地址并转发用户请求至相应的账户、容器或者对象服务；由于采用无状态的 REST 请求协议，可以进行横向扩展来均衡负载。

认证服务（Authentication Server）：验证访问用户的身份信息，并获得一个对象访问令牌（Token），在一定的时间内会一直有效；验证访问令牌的有效性并缓存下来直至过期时间。

缓存服务（Cache Server）：缓存的内容包括对象服务令牌，账户和容器的存在信息，但不会缓存对象本身的数据；缓存服务可采用 Memcached 集群，Swift 会使用一致性散列算法来分配缓存地址。

账户服务（Account Server）：提供账户元数据和统计信息，并维护所含容器列表的服务，每个账户的信息被存储在一个 SQLite 数据库中。

容器服务（Container Server）：提供容器元数据和统计信息，并维护所含对象列表的服务，每个容器的信息也存储在一个 SQLite 数据库中。

对象服务（Object Server）：提供对象元数据和内容服务，每个对象的内容会以文件的形式存储在文件系统中，元数据会作为文件属性来存储，建议采用支持扩展属性的 XFS 文件系统。

复制服务（Replicator）：会检测本地分区副本和远程副本是否一致，具体是通过对比散列文件和高级水印来完成，发现不一致时会采用推式（Push）更新远程副本，例如对象复制服务会使用远程文件拷贝工具 rsync 来同步；另外一个任务是确保被标记删除的对象从文件系统中移除。

更新服务（Updater）：当对象由于高负载的原因而无法立即更新时，任务将会被序列化到在本地文件系统中进行排队，以便服务恢复后进行异步更新；例如成功创建对象后容器服务器没有及时更新对象列表，这个时候容器的更新操作就会进入排队中，更新服务会在系统恢复正常后扫描队列并进行相应的更新处理。

审计服务（Auditor）：检查对象，容器和账户的完整性，如果发现比特级的错误，文件将被隔离，并复制其他的副本以覆盖本地损坏的副本；其他类型的错误会被记录到日志中。

账户清理服务（Account Reaper）：移除被标记为删除的账户，删除其所包含的所有容器和对象。

OpenStack块存储服务-cinder

cinder是openstack中提供块存储服务的组件，主要是为虚拟机实例提供虚拟磁盘。通过某种协议（SAS,SCSI,SAN,iSCSI等）挂接裸硬盘，然后分区、格式化创建的文件，或者直接使用裸硬盘存储数据的方式叫做块存储，每个裸硬盘通常也叫做Volume(卷）

Cinder架构图
![image](https://user-images.githubusercontent.com/36918717/177003351-ac602004-a937-4536-b677-4518cac197d8.png)

Cinder 包含如下几个组件：

cinder-api

接收 API 请求，调用 cinder-volume 执行操作。

cinder-volume

管理 volume 的服务，与 volume provider 协调工作，管理 volume 的生命周期。运行 cinder-volume 服务的节点被称作为存储节点。

 

cinder-scheduler

scheduler 通过调度算法选择最合适的存储节点创建 volume。

 

volume provider

数据的存储设备，为 volume 提供物理存储空间。

cinder-volume 支持多种 volume provider，每种 volume provider 通过自己的 driver 与cinder-volume 协调工作。

 

Message Queue

Cinder 各个子服务通过消息队列实现进程间通信和相互协作。因为有了消息队列，子服务之间实现了解耦，这种松散的结构也是分布式系统的重要特征。

Database Cinder 有一些数据需要存放到数据库中，一般使用 MySQL。数据库是安装在控制节点上的，比如在我们的实验环境中，可以访问名称为“cinder”的数据库。

物理部署方案

Cinder 的服务会部署在两类节点上，控制节点和存储节点。

我们来看看控制节点 devstack-controller 上都运行了哪些 cinder-* 子服务。

cinder-api 和 cinder-scheduler 部署在控制节点上，这个很合理。

至于 cinder-volume 也在控制节点上可能有些同学就会迷糊了：cinder-volume 不是应该部署在存储节点上吗？

要回答这个问题，首先要搞清楚一个事实：

OpenStack 是分布式系统，其每个子服务都可以部署在任何地方，只要网络能够连通。

无论是哪个节点，只要上面运行了 cinder-volume，它就是一个存储节点，当然，该节点上也可以运行其他 OpenStack服务。

cinder-volume 是一顶存储节点帽子，cinder-api 是一顶控制节点帽子。在我们的环境中，devstack-controller 同时戴上了这两顶帽子，所以它既是控制节点，又是存储节点。当然，我们也可以用一个专门的节点来运行 cinder-volume。

 

这再一次展示了 OpenStack 分布式架构部署上的灵活性：

可以将所有服务都放在一台物理机上，用作一个 All-in-One 的测试环境；而在生产环境中可以将服务部署在多台物理机上，获得更好的性能和高可用。

RabbitMQ 和 MySQL 通常是放在控制节点上的。

OpenStack 数据库服务-trove服务

Openstack Trove是openstack为用户提供的数据库即服务(DBaaS)。所谓DBaaS，即trove既具有数据库管理的功能，又具有云计算的优势。使用trove，用户可以：

 

"按需"获得数据库服务器

配置所获得的数据库服务器或者数据库服务器集群

对数据库服务器或者数据库服务器集群进行自动化管理

根据数据库的负载让数据库服务器集群动态伸缩

与openstack的其他组件一样，trove也提供RESTful API，并通过RESTful API和其他组件进行交互。

Trove架构图：

![image](https://user-images.githubusercontent.com/36918717/177003362-f8ba0b87-0a0d-4fc9-83ad-b696c6c53e1f.png)
Trove API和用户进行交互，当Trove API接收到用户请求时，trove API首先会调用Keystone的API来对用户进行认证，认证通过后才会去执行相应的操作。Trove API会同步处理操作简单的一些请求，复杂的请求则会通过Message Queue (RabbitMQ)交给Task Manager来处理。Task Manager会监听RabbitMQ的一个topic，收到请求后就会进行处理。这些请求通常是分配数据库实例、管理数据库实例的生命周期、操作数据库等。和openstack的其他组件一样，Trove也有一个Infrastructure Database来存储自己本身的数据，如数据库实例的信息等。Trove conductor的主要功能是接收来自guest agent的状态更新信息，这些信息会被存储在Infrastructure database里面或者作为调用结果返回给其他服务。通常这些状态更新信息包括：guest agent心跳包，数据库备份状态等。Guest Agent运营于数据库服务器中(虚拟机)，给trove其他组件提供了一套内部使用的API，trove的其他组件通过Message Queue来调用这些API，guest agent收到API调用请求后，执行相应的数据库操作。

 

Trove包含的组件以及功能

trove-api

用于操作请求的接收和分发操作提供REST风格的API，同时与trove-conductor和trove-taskmanager通信，一些轻量级的请求比如获取实例状态，实例数量等操作都是自身直接处理或访问trove

conductor和trove-taskmanager处理，比较重量级的操作比如创建数据库，创建备份等操作都是通过rpc传递给trove-taskmanager，taskmanager，然后在通过调用nova、swift、neutron、cinder等组件来完成操作。

trove-conductor

将vm内trove-guestagent发送的状态信息保存到数据库，与trove-guestagent的通信是通过rpc来实现的，trove-conductor这个组件的目的是为了避免创建的数据库的实例直接访问数据库，它是做为一个trove-guestagent将昨天写入数据库的中间件。

trove-taskmanager

执行trove中大部分复杂的操作，请求者发送消息到task manager，task manager在请求者的上下文中调用相应的程序执行这些请求。task manager处理一些操作，包括实例的创建、删除，与其他服务如Nova、Cinder、Swift等的交互，一些更复杂的Trove操作如复制和集群，以及对实例的整个生命周期的管理。trov-taskmanager就像是其他openstak服务的客户端，如nova，swift，cinder等，当要创建数据库实例时就将请求发送给nova，让nova去创建个实例，要备份的话就调用swift接口上传备份。

trove-guestagent

trove-guestagent集成在vm镜像里面，通过监听rpc里面task manager发过来的指令，并在本地执行代码完成数据库任务，taskmanager将消息发送到guest agent，guest agent通过调用相应的程序执行这些请求。

 

OpenStack计算服务-nova

Nova 是 OpenStack 最核心的服务，负责维护和管理云环境的计算资源。OpenStack 作为 IaaS 的云操作系统，虚拟机生命周期管理也就是通过 Nova 来实现的。

功能：

实例生命周期管理

管理计算资源

网络和认证管理

REST风格的API

异步的一致性通信

Hypervisor透明：支持Xen,XenServer/XCP, KVM, UML, VMware vSphere and Hyper-V

Nova架构图
![image](https://user-images.githubusercontent.com/36918717/177003374-93b95756-959b-46ce-8c1e-cf1d6b6bcac6.png)

Nova组件：

1、Nova API ：HTTP服务，用于接收和处理客户端发送的HTTP请求

2、Nova Compute ：

- Nova组件中最核心的服务，实现虚拟机管理的功能。

实现了在计算节点上创建、启动、暂停、关闭和删除虚拟机、虚拟机在不同的计算节点间迁移、虚拟机安全控制、管理虚拟机磁盘镜像以及快照等功能。

3、Nova Cert ：用于管理证书，为了兼容AWS。AWS提供一整套的基础设施和应用程序服务，使得几乎所有的应用程序在云上运行;

4、Nova Conductor ：RPC服务，主要提供数据库查询功能。

5、Nova Scheduler ：Nova调度子服务。当客户端向Nova 服务器发起创建虚拟机请求时，决定将续集你创建在哪个节点上。

6、Rabbit MQ Server ：

- OpenStack 节点之间通过消息队列使用AMQP（Advanced Message Queue Protocol）完成通信。

Nova 通过异步调用请求响应，使用回调函数在收到响应时触发。因为使用了异步通信，不会有用户长时间卡在等待状态。

7、Nova Console、Nova Consoleauth、Nova VNCProxy ：Nova控制台子服务。功能是实现客户端通过代理服务器远程访问虚拟机实例的控制界面。

8、nova-volume ：是创建、挂载、卸载持久化的磁盘虚拟机，运行机制类似nova-computer。

同样是接受消息队列中的执行指令，并执行相关指令。

volume相关职责包括：创建硬盘、删除硬盘，弹性计算硬盘，为虚拟机增加块设备存储。

Nova工作流程：
![image](https://user-images.githubusercontent.com/36918717/177003382-80200ad1-e339-4285-8592-d40ab48451df.png)
下面以创建虚拟机为例：

 

调用nova-api创建虚拟机接口，nova-api对参数进行解析以及初步合法性校验

调用compute-api创建虚拟机vm接口，computer-api根据虚拟机参数（cpu、内存、磁盘、网络、安全组等）信息，访问数据库创建数据模型虚拟机实例记录。

computer-api通过rpc的方式将创建虚拟机的基础信息封装成消息发送至消息中间件指定消息队列“scheduler”

nova-scheduer订阅了消息队列“scheduler”的内容，接受到创建虚拟机的消息后，进行过滤，根据请求的虚拟资源，即flavor的信息,选择一台物理主机部署，如novan1，nova-scheduler将虚拟机的基本信息、所属物理主机信息发送值消息中间件指定消息队列“computer.novan1”

novan1上nova-compute守护进程订阅消息队列“computer.novan1”，接受到消息后，根据虚拟机基本信息开始创建虚拟机

nova-computer调用network-api分配网络ip

nova-network接受到消息后，从fixedIP表中拿出一个可用的IP，nova-network根据私网资源池，结合DHCP，实现ip分配和ip绑定

nova-computer通过调用volume-api实现存储划分，最后调用底层虚拟化技术，部署虚拟机。

 

 

OpenStack 镜像服务-glance

Glance（OpenStack Image Service）是一个提供发现，注册，和下载镜像的服务。Glance 提供了虚拟机镜像的集中存储。通过 Glance 的 RESTful API，可以查询镜像元数据、下载镜像。虚拟机的镜像可以很方便的存储在各种地方，从简单的文件系统到对象存储系统(比如 OpenStack Swift)。

 

在 Glance 里镜像被当做模板来存储，用于启动新实例。Glance 还可以从正在运行的实例建立快照用于备份虚拟机的状态。

 

Glance 具体功能如下：

提供 RESTful API 让用户能够查询和获取镜像的元数据和镜像本身；

支持多种方式存储镜像，包括普通的文件系统、Swift、Ceph 等；

对实例执行快照创建新的镜像。

 

Glance架构
![image](https://user-images.githubusercontent.com/36918717/177003395-bbf4ede3-6c91-4689-9a00-a3dc14a0936e.png)

组件介绍：

glance-api

glance-api是系统后台运行的服务进程，对外提供REST API,响应image查询，获取和存储的调用，glance-api不会真正处理请求。

如果是与image metadata相关的操作，glance-api会把强求转发给glance-registry;如果是与image自身存取的相关操作，glance-api会把请求转发给image的store backend.



glance-registry

glance-registry是系统后台运行的服务进程，负责处理和存取image的metadata，例如image的大小和类型

 

glance-db

image的metadata会保持到database中，默认是mysql

Store adapter（后端存储）

 

glance 自己并存储image，真正的image是存放在backend中的。glance支持多种backend

上传镜像流程：

1.  管理员使用上传镜像命令。Glance-API服务收到请求，并通过它的中间件，解析出版本号等信息。

2,3.  Glance-Registry服务的api获取一个registry client，调用registry client的add_image函数。此时镜像的状态为“queued”， 标识该镜像ID已经被保留，但是镜像还未上传。

4.  Glance-Registry服务执行client的add_image函数，向glance数据库中insert一条记录。

5，6,7.   Glance-API调用Glance-Registry的update_image_metadata函数，更新数据库中该镜像的状态为“saving”，标识镜像正在被上传。

8.  Glance-API调用后端存储接口提供的add函数上传镜像文件。 

9,10,11.  Glance-API调用Glance-Registry的update_image_metadata函数，更新数据库中该镜像的状态为 “active”并发通知。“active”标识镜像在Glance中完全可用。

我们知道glance是管理镜像的，那么针对IAAS来说，VM镜像、DB（MySQL、mariadb）、NoSQL（redis、kafka、rabbitmq、influxDB）等基础服务的申请和部署都需要事先准备好对应的镜像，这些镜像就可以使用glance管理。

 

OpenStack网络服务-neutron（算是整个OpenStack最复杂部分之一）

OpenStack网络服务提供了一个API接口，允许用户在云上设置和定义网络连接和地址。这个网络服务的项目代码名称是Neutron。OpenStack网络处理虚拟设备的创建和管理网络基础设施，包括网络、交换机、子网以及由计算服务（nova）管理的设备路由器。高级服务，如防火墙或虚拟私人网络（VPN）也可以使用。

 

OpenStack网络由neutron-server，持久化存储数据库，和任意数量的插件代理组成，这些代理提供其他服务，如与本地linux联网接口机制、外部设备或SDN控制器。

 

OpenStack网络是完全独立的，可以部署到一个专用主机。如果你的部署使用了一个控制器主机运行集中计算组件，你可以部署网络服务来取代主机的设定。

Neutron架构：

![image](https://user-images.githubusercontent.com/36918717/177003399-62ce4913-b91e-49ca-93e5-9460b5654674.png)
Neutron组件部分：

neutron-server

openstack各大组件都向外提供API服务，neutron也不例外，有neutron-server跑在controller节点，提供标准的api，还有各种extension实现api功能的扩展。ML2 plugin作为核心插件实现api中的network，subnet，port和security_group，还支持一些extension，主要实现layer 2转发的功能。高级services的plugin实现更多的高级功能，如l3_router实现layer 3的功能，主要是三层转发和NAT的功能，其它plugin还能实现LB，FW和VPN的功能。

neutron-server DB层是做什么的？

neutron-server肯定得把network,subnet,port等相关的信息存放在数据库中，还得定义数据存放的格式和数据之间的关系，以及新加属性后的升级等，DB层提供的功能保证数据操作是安全和一致性的，总之DB层是通用层，DB操作交给这层就行了，不用其它层操心。

plugin是什么玩意？

plugin继承DB层定义的类和方法，然后实现对agent的notify和RpcCallback。api请求来之后先对数据库进行操作，然后再通知agent进行动作，或者收到agent rpc消息进行相应的操作。

ML2 plugin又是干什么的？

ML2= multiple layer 2，实现了多种多样的layer 2转发类型，如flat，vlan，VXLAN和GRE，同样实现vlan转发既可以用ovs也可以用linux bridge，所以又搞出个type_driver和mechanisim_driver。typedriver是抽象层面的概念，每种类型的网络都是不一样的，表示的信息不一样，如vlan网络就是vlan id了，vxlan网络就是vni了，所以一种type_driver表示一种网络，创建这种类型的网络就用对应的type_driver做检查和分配信息，如vlan id是否够用，分配一个vlan id或者指定的vlan id是否已经被占用。mechanisim_driver可以认为是实现机制，也可以认为它是ML2的plugin，这个ML2的plugin可以和对应的l2 agent通信，linux bridge agent可以实现基于vlan的转发，ovs agent也可以实现基于vlan的转发，那么linux bridge和ovs就是两种实现机制，所以每种实现机制需要向neutron-server报告自己支持的网络类型和接口类型，还有是否支持security_group等。

service plugin又能干什么？

service plugin就是更高级功能的plugin，包括l3_router plugin实现的是三层转发和NAT功能。service plugin也分为type和provider，类比ML2 plugin，type就是routing，firewall, vpn和lb，privider就是driver，同样是routing，可以用l3 agent的namespace实现转发，也可以用H3C的硬件来实现，只要H3C提供plugin就行了。

agent

agent运行在compute node或者network node，是具体干活的，控制数据面转发机制，把流量打通，如已经用到的dhcp-agent, linux-bridge-agent，l3-agent，openvswitch-agent，metadata-agent等等，agent和neutron-server侧的plugin进行交互。

neutron-server和agent通信机制

neutron原生实现agent和neutron-server之间用消息队列通信，neutron-server可以push，agent进行pull，或者agent定时信息同步，然后agent调用命令搭建数据面的路径。agent也可以是SDN controller，厂商自己实现的plugin通过其它方式和controller通信，对接了SDN controller就不再需要原生的那些agent了，如H3C vcf和contrail。

以创建一个 VLAN100 的 network 为例，假设 network provider 是 linux bridge，流程如下：

1、Neutron Server 接收到创建 network 的请求，通过 Message Queue（RabbitMQ）通知已注册的 Linux Bridge Plugin。

2、Plugin 将要创建的 network 的信息（例如名称、VLAN ID等）保存到数据库中，并通过 Message Queue 通知运行在各节点上的 Agent。

3、Agent 收到消息后会在节点上的物理网卡（比如 eth2）上创建 VLAN 设备（比如 eth2.100），并创建 bridge （比如 brqXXX） 桥接 VLAN 设备。

 

OpenStack大数据计算服务-sahara

sahara（以前叫savanna）以前是openstack的孵化项目，但是从openstack的下一个版本juno开始将成为openstack的核心项目，它是由领先的Apache Hadoop贡献方Hortonworks公司，最大的OpenStack 系统集成商Mirantis公司，以及全球领先的开源解决方案及最新版OpenStack的最大贡献方红帽公司联合发起的。Sahara项目旨在为OpenStack用户提供一种简单、快捷地部署以及管理Hadoop集群的方案，类似于亚马逊Elastic MapReduce (EMR) 服务                 

Sahara架构：

![image](https://user-images.githubusercontent.com/36918717/177003409-1f1f43a6-ad64-438d-8abc-540f7cadf633.png)

Sahara组件：

主要包括以下几个组件：

1、Auth component（认证组件） - 负责和认证服务交互完成客户认证。

2、DAL - 数据访问层, 负责为持久化数据提供数据库访问接口。

3、Secure Storage Access Layer（安全存储访问层） - 保存用户认证信息，比如用户密码、密钥等。

4、Provisioning Engine - 该组件负责和Openstack其他组件交互，比如Nova组件、Heat组件、Cinder组件以及Glance组件等。

5、Vendor Plugins（厂商插件） - 负责配置和启动计算框架，不同的计算框架启动方式和配置都不一样，因此提供插件机制，使sahara同时可支持多种计算框架。已经完成集成的插件包括Apache Ambari和Cloudera Management Console等。

6、EDP - Elastic Data Processing，负责调度和管理任务。

7、REST API - 通过REST HTTP API接口暴露sahara管理功能。

8、Python Sahara Client - sahara命令行管理工具。

9、Sahara pages - Openstack Dashboard显示页面。

 

OpenStack鉴权服务-keystone

Keystone是 OpenStack中负责管理身份验证、服务规则和服务令牌功能的模块。用户访问资源需要验证用户的身份与权限，服务执行操作也需要进行权限检测，这些都需要通过 Keystone来处理。Keystone类似一个服务总线，其他服务通过keystone来注册其服务的Endpoint（服务访问的URL），任何服务之间相互的调用，需要经过Keystone的身份验证，来获得目标服务的Endpoint来找到目标服务。

Keystone架构
![image](https://user-images.githubusercontent.com/36918717/177003415-d8144860-8aa6-4006-889c-714fe33b9819.png)

Keystone组件：

1、Keystone API

Keystone API与Openstack其他服务的API类似，也是基于ReSTFul HTTP实现的。

Keystone API划分为Admin API和Public API：

Public API不仅实现获取版本以及相应扩展信息的操作，同时包括获取Token以及Token租户信息的操作；

Admin API主要提供服务开发者使用，不仅可以完成Public API的操作，同时对User、Tenant、Role和Service Endpoint进行管理操作。

 

2、Router

Keystone Router主要实现上层API和底层服务的映射和转换功能，包括四种Router类型。

（1） AdminRouter

负责将Admin API请求映射为相应的行为操作并转发至底层相应的服务执行；

（2） PublicRouter

与AdminRouter类似；

（3） PublicVersionRouter

对系统版本的请求API进行映射操作；

（4） AdminVersionRouter

与PublicVersionRouter类似。

3、Services

Keystone Service接收上层不同Router发来的操作请求，并根据不同后端驱动完成相应操作，主要包括四种类型；

（1） Identity Service

Identity Service提供关于用户和用户组的授权认证及相关数据。

Keystone-10.0.0支持ldap.core.Identity,Sql.Identity两种后端驱动，系统默认的是Sql.Identity；

 users和groups

 （2） Resource Service

Resouse服务提供关于projects和domains的数据

 projects和domains

 在v3版本中的唯一性概念

（3） Assignment Service

Assignment Service提供role及role assignments的数据

 roles和role assignments

（4） Token Service

Token Service提供认证和管理令牌token的功能，用户的credentials认证通过后就得到token

Keystone-10.0.0对于Token Service

支持Kvs.Token，Memcache.Token，Memcache_pool.Token,Sql.Token四种后端驱动，系统默认的是kvs.Token

（5） Catalog Service

Catalog Service提供service和Endpoint相关的管理操作（service即openstack所有服务，endpont即访问每个服务的url）

keystone-10.0.0对Catalog Service支持两种后端驱动：Sql.Catalog、Templated.Catalog两种后端驱动，系统默认的是templated.Catalog；

（6） Policy Service

Policy Service提供一个基于规则的授权驱动和规则管理

 

keystone-10.0.0对Policy Service支持两种后端驱动：rules.Policy,sql.Policy，默认使用sql.Policy

4、Backend Driver

Backend Driver具有多种类型，不同的Service选择不同的Backend Driver。

Keystone流程：

![image](https://user-images.githubusercontent.com/36918717/177003425-fb4ff81d-5676-4e52-81cb-96b5fc4922fe.png)

例如

1.用户/API 想创建一个实例，首先会将自己的credentials发给keystone。认证成功后，keystone会颁给用户/API一个临时的令牌(Token)和一个访问服务的Endpoint。PS:Token没有永久的，默认有效期是24h

2.用户/API 把临时Token提交给keystone，keystone并返回一个Tenant(Project)

3.用户/API 向keystone发送带有特定租户的凭证，告诉keystone用户/API在哪个项目中，keystone收到请求后，会发送一个项目的token到用户/API PS：第一个Token是来验证用户/API是否有权限与keystone通信，第二个Token是来验证用户/API是否有权限访问我keystone的其它服务。用户/API 拿着token和Endpoint找到可访问服务

4.服务向keystone进行认证，Token是否合法，它允许访问使用该服务（判断用户/API中role权限）？

5.keystone向服务提供额外的信息。用户/API是允许方法服务，这个Token匹配请求，这个Token是用户/API的

6.服务执行用户/API发起的请求，创建实例

7.服务会将状态报告给用户/API。最后返回结果，实例已经创建

文章结尾我们以新建CVM（cloud virtual machine）为例，熟悉下OpenStack创建云主机的流程。
![image](https://user-images.githubusercontent.com/36918717/177003433-5a7c09d4-0db4-4d9e-ae40-be9271ef1a73.png)
1、访问dashboard，输入用户名和密码点击登录后，表单信息传给keystone进行验证

2、keystone接收到前端表单传过来的域、用户、密码信息以后，查询了数据库，确认身份后将一个token（就像是办了身份证～～～）返回给该用户，让这个用户以后再进行操作的时候不需要再提供账号密码，而是拿出token来

3、horizon拿到token之后，实际上这里在web页面上的显示就是登录成功了，接着找到创建云主机的按钮并点击，填写云主机相关配置信息。填好信息之后，点击启动实例按钮，horizon将创建云主机的请求、云主机相关配置信息以及刚刚keystone返回给的token传给nova-api。

4、nova-api收到信息之后，带着3给的token请求keystone验证是否合法

5、kestone收到nova-api传来的token并验证后返回nova-api

6、验证通过后，nova-api将主机配置信息写到DB

7、数据库把配置信息收好之后，对nova-api说了声，我放好了

8、nova-api将创建主机信息写入消息队列（并连带在DB的配置信息）

9、nova-scheduler从消息队列获取消息

10、nova-scheduler解析之后发现是创建主机，则访问DB获取主机配置信息

11、DB收到请求之后将主机信息返回给nova-scheduler

12、nova-scheduler拿到云主机配置信息之后，使用调度算法之后决定了让nova-compute去干这个事情，就将创建云主机任务写到消息队列

13、nova-compute从消息队列获取消息之后，本应该直接去数据库拿取配置信息，但因为nova-compute的特殊身份，nova-compute所在计算节点上全是云主机，万一有一台云主机被黑客入侵从而控制计算节点，直接拖库是很危险的。所以不能让nova-compute知道数据库在什么地方

14、nova-compute没办法去数据库取东西难道就不工作了吗？那可不行啊，他不知道去哪取，但他哥们知道啊，于是nova-compute向消息队列发送消息告诉nova-conductor去DB获取信息

15、nova-conductor从消息队列获取消息

16、nova-conductor向DB发送请求，获取对应云主机的配置信息

17、DB把云主机配置信息返回至nova-conductor

18、nova-conductor将云主机配置信息写入消息队列

19、nova-compute从消息队列获取云主机配置信息

20、nova-compute拿到云主机配置信息之后，先访问glance-api获取对应的主机镜像

21、glance-api获取nova-compute的请求之后，拿着nova-compute的token访问keystone，keystone进行验证并返回给glance-api

22、验证通过后，glance-api将镜像配置信息返回给nova-compute

23、接着nova-compute找到neutron-server，获取对应的网络资源

24、neutron-server也将带着nova-compute的token信息访问keystone进行验证，并返回给neutron-server

25、验证通过后，neutron-server将网络资源返回给nova-compute

26、nova-compute请求cinder-api要存储资源，例如硬盘

27、cinder-api带着nova-compute的token请求keystone进行验证，并返回给cinder-api

28、验证通过后，cinder-api将存储资源信息返回给nova-compute

29、nova-compute获取所有资源之后，将工作交给hypervisor创建VM (kvm，zen等虚拟化技术）。

 

 

总结：以上介绍了OpenStack的整体架构，以及几个核心组件的作用和组成部分。OpenStack作为一个开源的云计算框架，不仅为各大云厂商提供了指导意义，同时给学习云计算的同学也提供了学习和借鉴的地方。不过，OpenStack虽然看上去NB，但由于各个组件都是不同厂家提供的，因此在服务之间的并发性、稳定性等可能不是非常理想，网上有用户提出原生OpenStack支持500台以下云主机，这个数字对于当下的云厂商可能远远不够。因此像AWS、腾讯（T-stack）、阿里（Apsara ）都以自研为主；京东、携程、小米等基本上用的都是OpenStack（可能有些组件自己加以改造）。无论是自研还是使用开源，核心思想几乎是一样的。

OpenStack是一个庞大的开源项目，几乎每个组件都能出一本厚厚的书（有些部分已经有出对应的书籍）。

接下来可能需要找几台机器搭建一个测试集群玩玩，搭建集群我还是比较偏向于手工搭建，虽然网上有一些自动化搭建工具（自动化工具一般都是留给非挨踢攻城狮使用），但不建议使用，尤其是对想深入玩OpenStack的同学。因为徒手搭建不仅能了解每个组件的具体使用情况，还能踩到一些坑（不踩到足够多的坑，你无法拿下OpenStack），而且OpenStack各个组件基本上都是python实现，非编译型的（不像docker、K8S、kernel这些编译型项目），直接部署安装即可。

最后就是八卦一下OpenStack的发起者NASA，不错，就是美国的那个NASA，是一个政府机构，不过目前已经跑路了，什么意思？意思就是不再主导和支持OpenStack这个开源项目了，为什么呢？答案很简单：政府部门，追求的是用最低的成本做更多的事情，OpenStack是个持续投入的开源项目，是个无底洞，政府不可能作为开发人员去投入成本的。目前NASA已经成了使用者。

 

参考资料

https://docs.openstack.org/

https://blog.51cto.com/13914991/2300918

https://docs.openstack.org/install-guide/get-started-logical-architecture.html

https://blog.51cto.com/13643643/2171262?cid=719235

https://blog.csdn.net/li575098618/article/details/49818193

https://www.cnblogs.com/gaozhengwei/p/7097605.html

https://blog.csdn.net/jackliu16/article/details/79969927

https://blog.csdn.net/dylloveyou/article/details/80698420

https://blog.csdn.net/south_l/article/details/81020974

https://blog.csdn.net/jessica_ff/article/details/81982214

https://blog.51cto.com/11555417/2345634

https://blog.csdn.net/dylloveyou/article/details/80530838

https://blog.csdn.net/hzrandd/article/details/9980405

https://www.cnblogs.com/tongxiaoda/p/8862789.html

https://blog.51cto.com/arkling/2289431

https://blog.csdn.net/cloudresearch/article/details/22621409 https://www.cnblogs.com/int32bit/p/5395842.html

http://www.pianshen.com/article/9495385086/

https://www.cnblogs.com/ztxd/p/12227867.html

https://www.cnblogs.com/awmpy/p/6637869.html

 





























