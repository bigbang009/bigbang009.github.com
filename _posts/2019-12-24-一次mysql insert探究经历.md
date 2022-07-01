---
layout:     post
title:      mysql
subtitle:   一次mysql insert探究经历
date:       2019-12-24
author:     lwk
catalog: true
tags:
    - linux
    - mysql
---
一次mysql insert探究经历

用户反馈使用A系统安装包,安装成功了却没有实例信息,是怎么回事呢?

根据经验,遇到安装成功或升级成功没有实例的问题,一般有以下几个原因:

1、  实例上报堵塞

2、  包安装路径配置错误

3、  用户机器连不上A系统管理机器（实例上报通过反向扫描机器上所有A系统包汇总后通过HTTP上报至A系统机器）

那就从这几个方面查起吧。

实例上报拥堵了？

A系统系统也有自身监控，实例上报堵塞会有对应告警信息，并且从监控视图查看实例上报并没有堵塞，排除其可能性
![image](https://user-images.githubusercontent.com/36918717/176820247-3f6fdbde-6bff-475f-af45-45d208a2d5d2.png)
用户的包安装路径配置错误？

目前A系统上报包实例信息只会扫描固定几个目录下（/usr/local/、/usr/local/app、/usr/local/services、/home/oicq）【这篇文章不讨论织云架构设计】，如果用户包配置安装路径install_path不在这几个范围之内，则可能出现安装后无实例。

看下用户的包配置安装路径
![image](https://user-images.githubusercontent.com/36918717/176820273-77f3bc9b-b515-4fc9-b3ba-24dfc442f7d4.png)

Install_path为空，说明用户没自定义安装路径，安装时会以install_base为准，install_base在上面所列的扫描范围内，也是没问题的。排除其可能性。

用户机器连不上A系统管理机器？

需要登录用户机器排查
![image](https://user-images.githubusercontent.com/36918717/176820294-f3538e43-ed98-45a9-8366-dec3306b2744.png)

执行/tmp/pkg_tools_v2/scan_pkg/scan_packages_info.sh看有没有报错，没报错表示正常。

执行后也没报错，那就是正常的了？还是放心，把上报结果也打印出来看下。
![image](https://user-images.githubusercontent.com/36918717/176820321-311a9487-c0ec-4043-9adf-3d6f716490e2.png)

从结果看有将用户包统计上报。

不放心，再strace看，没异常
![image](https://user-images.githubusercontent.com/36918717/176820342-bf8c0df7-a9b5-44e2-8bd5-49850d2de171.png)

![image](https://user-images.githubusercontent.com/36918717/176820349-cfc1cc93-7839-45fe-96f4-28df31cda49e.png)

还是不放心，再抓包分析下
![image](https://user-images.githubusercontent.com/36918717/176820369-8ec522aa-9322-4b0b-b8ee-e3f6cec193d4.png)

从客户端和服务端抓包分析，发送和接收都正常，而且包体也正确。

奇怪了，包没问题，实例上报也没问题，是哪里的问题呢？

这个时候只能去查服务端处理实例上报的逻辑了，即pkgScanWorker。查之前先看下pkg实例上报的逻辑图：
![image](https://user-images.githubusercontent.com/36918717/176820393-f83e3edb-74f2-4782-98b4-2fc3e2191d7e.png)

总体逻辑就是包操作（安装、升级、卸载、重启等）触发现网包实例上报至统一接入，再转到接口机，接口机将实例数据push到rabbitmq，pkgscanworker异步从rabbitmq取数据进行处理入库。

首先最直接的就是查日志
![image](https://user-images.githubusercontent.com/36918717/176820416-33eaa41b-98f3-4cce-875f-9e6bfa58a69f.png)


从日志看到addPkgInstance返回为空，看下代码对应位置：

![image](https://user-images.githubusercontent.com/36918717/176820432-98a91823-847d-49d6-84ad-9ab7cb5e1132.png)

![image](https://user-images.githubusercontent.com/36918717/176820444-edf3962d-7691-4e09-b0fd-7852a92a5d92.png)

报错：查询出错 (1062): Duplicate entry'10.62.158.14-/qidian/TSF2_QDA-/usr/local/services/TSF2_QDA-1.0' for key 'ip_2'

字面意思就是插入了重复数据。

重复数据？难道实例数据库已经有了这条记录？
![image](https://user-images.githubusercontent.com/36918717/176820467-6dc66d39-a43d-41b5-97f7-b78531c92cdb.png)

从上面记录看，并没有10.**.**.14对应TSF2_QDA包的实例记录（对于之前遇到过这类问题或熟悉mysql的同学已经知道问题所在）。

10.**.**.14-/qidian/TSF2_QDA-/usr/local/services/TSF2_QDA-1.0对应的column是ip+packagePath+installPath三个字段组成。查看一下表结构索引
![image](https://user-images.githubusercontent.com/36918717/176820507-92d5f58d-2994-4577-92ad-25877d08eba0.png)
有个UNIQUE，而且是ip+packagePath+installPath组成。怀疑和这个唯一索引有关系。虽然存在TSF2_qda记录，但对应字段是小写的，本次插入记录是TSF2_QDA（大写），理论上不会冲突。难道UNIQUE不区分大小写？

 

下载5.5版本的代码，编译后，启动mysql，并创建test数据库，并创建unique index

 

wget https://downloads.mysql.com/archives/get/file/mysql-5.5.59.tar.gz

BUILD/compile-pentium-debug --prefix=/usr/local/mysql-5.5.59/

make && make install
![image](https://user-images.githubusercontent.com/36918717/176820539-2cafda4f-e5fd-41fd-8067-6e8c5d291cf3.png)

gdb调试mysql insert过程（前期做了一些功课，所以直接在storage/innobase/handler/ha_innodb.cc的innobase_mysql_cmp函数设置断点，这里会根据collation对预插入数据进行处理）

![image](https://user-images.githubusercontent.com/36918717/176820555-07c02118-5406-40f4-803f-b905c539dae2.png)

![image](https://user-images.githubusercontent.com/36918717/176820563-7dd64de4-0741-4136-9ee1-28e17e6f9a37.png)


![image](https://user-images.githubusercontent.com/36918717/176820575-b1101ea5-327d-44ce-a8ce-8f4ed0c11e57.png)


charset_number在strings/ctype-extra.c定义如下：



![image](https://user-images.githubusercontent.com/36918717/176820588-c4bc5369-4235-42d0-af9d-2da80797814f.png)

![image](https://user-images.githubusercontent.com/36918717/176820592-bdd2d64b-401f-47a8-bddc-0f509130a077.png)

![image](https://user-images.githubusercontent.com/36918717/176820600-df41a9ab-e2b1-4d21-b285-a8a21418211f.png)

![image](https://user-images.githubusercontent.com/36918717/176820608-c326379b-ba71-41a8-80ac-38730598b23e.png)

在没gdb之前，我猜测mysql是将待插入数据和db中已有的数据进行比较，例如待插入数据是insert into test values('lwK', '/home/oicq/Test');

将lwK-/home/oicq/Test拼接后和已有的拼接记录进行比较。最后发现，想法太天真了，为啥呢，因为太简单且低效。从gdb看不是简单的数据拼接比较，而是使用的B树（请自己google，不赘述）。

综上，mysql insert之前会对待插入数据根据collation，与已有的record进行比较，

当collation是latin1_general_ci则会忽略大小写。

Mysql_insert操作源码在sql_insert.cc
![image](https://user-images.githubusercontent.com/36918717/176820627-dd10d768-e134-46c2-97e1-a4331adf7219.png)

![image](https://user-images.githubusercontent.com/36918717/176820636-15a85063-5c09-414d-8c4a-4c7dfe296ef9.png)

![image](https://user-images.githubusercontent.com/36918717/176820646-23363860-69b3-4049-ad5c-cbb5dbf4dc15.png)

![image](https://user-images.githubusercontent.com/36918717/176820653-3ee069b1-dbcc-482a-849c-ec6a464afd04.png)

![image](https://user-images.githubusercontent.com/36918717/176820662-b72c9952-572f-4b0b-b413-dc5a269b9fff.png)

中间就是一系列的n（NEXT）操作


![image](https://user-images.githubusercontent.com/36918717/176820674-5267308c-a2a1-41e9-850b-76bd913f2f77.png)

执行到sql_connect.cc的870行时，进入到do_command函数体


![image](https://user-images.githubusercontent.com/36918717/176820693-808fc545-e650-4809-97db-ac0c0f78ac9a.png)

![image](https://user-images.githubusercontent.com/36918717/176820699-c7e1bacd-633b-4e31-9f31-fd9cc8a28b86.png)

![image](https://user-images.githubusercontent.com/36918717/176820711-26489b59-ba23-4f4a-93b3-9a30d8112581.png)

error=121，对应错误码定义在include/my_base.h文件，宏HA_ERR_FOUND_DUPP_KEY


![image](https://user-images.githubusercontent.com/36918717/176820730-324de428-5ba1-47d7-a29d-93e2d57df0d9.png)

![image](https://user-images.githubusercontent.com/36918717/176820737-f237c060-32ca-4610-989a-9e0d43d85189.png)

![image](https://user-images.githubusercontent.com/36918717/176820746-4ff3bc86-ed98-4817-8df5-2d22ea7095e2.png)

![image](https://user-images.githubusercontent.com/36918717/176820754-a2e9bde2-a32a-4b1b-9de5-aa6885a3320b.png)

![image](https://user-images.githubusercontent.com/36918717/176820762-ba92153f-eb72-43a7-921a-8ca498171971.png)

![image](https://user-images.githubusercontent.com/36918717/176820775-363cf232-830d-4c31-9448-ee8907f3a823.png)

![image](https://user-images.githubusercontent.com/36918717/176820785-667b8af5-bb22-4715-a05f-c5f329da9591.png)

![image](https://user-images.githubusercontent.com/36918717/176820800-94e3fb0e-9b5a-4a60-9ac9-ac0c83869972.png)

和插入失败提示错误一样。

 

现网包的实例数据库表结构里ip+packagePath+installPath构成了unique index。并且ip、packagePath、installPath对应的collation都是latin_general_ci，

经google才知道latin_general_ci中的ci表示case insensitive（大小写无关），cs结尾表示case sensitive（即大小写有关），详细内容参考https://juejin.im/post/5bfe5cc36fb9a04a082161c2
![image](https://user-images.githubusercontent.com/36918717/176820817-bdf99494-7cf6-40a5-b632-04ca1aaafec0.png)

当然，最终的解决方案也说下：解决方案不是改已有的表字段collation，而是修改pkgScanworker的select逻辑。

总结一下：负责该系统已经有两年多时间，协助用户排查问题的同时自身也在不断的提升（当然还有很大的提升空间），总结一句话80%的问题需要耗费80%的时间，但实际解决的话只需要耗费20%的时间，其实有时一个问题可能排查了很久，最终可能只要改几行代码，甚至不改代码都能解决，解决问题不是关键，关键是从问题本身深入挖掘出未知的知识以及对以后设计上的思考、提前规避已知的问题。
