---
layout:     post
title:      linux内核VFS之read
subtitle:   linux内核VFS之read
date:       2020-09-04
author:     lwk
catalog: true
tags:
    - linux
    - read
    - vfs
---
今天我们看下read过程。从系统调用，到读取文件内容是怎么样的过程。

![image](https://user-images.githubusercontent.com/36918717/177033305-0829a5f8-5d6f-4305-9c93-a9098ddfe848.png)
![image](https://user-images.githubusercontent.com/36918717/177033309-b5abe51a-aae6-4901-bc8d-c1ef5f0ee151.png)
接下来看下如何根据fd获取files

![image](https://user-images.githubusercontent.com/36918717/177033312-7eb75a22-e452-46b1-8b57-27e103dc5639.png)
![image](https://user-images.githubusercontent.com/36918717/177033317-3a61172e-899d-4614-95e5-47b05a21f352.png)
![image](https://user-images.githubusercontent.com/36918717/177033320-e63cf05b-73f6-4c31-8f66-4c623b4fa1eb.png)
![image](https://user-images.githubusercontent.com/36918717/177033323-b59817c3-804e-4101-b2f8-1eaf9d966551.png)
![image](https://user-images.githubusercontent.com/36918717/177033325-8710fdcc-2201-4e5d-8a13-81da7d1533a0.png)
![image](https://user-images.githubusercontent.com/36918717/177033327-032077d6-fdfa-47b4-85b5-60be2690cf04.png)
![image](https://user-images.githubusercontent.com/36918717/177033336-f53e6833-2b22-4d85-89a0-ecd1fa5cbaf9.png)
接下来看下__fcheck_files函数逻辑：
![image](https://user-images.githubusercontent.com/36918717/177033339-da7f34c7-a02c-4fd9-8888-10cb4cdbe490.png)

![image](https://user-images.githubusercontent.com/36918717/177033341-cdc144b7-fd54-4988-9f09-3186a5f80d41.png)
![image](https://user-images.githubusercontent.com/36918717/177033343-636732c5-6425-45a7-b0ae-36ffd5be8034.png)
![image](https://user-images.githubusercontent.com/36918717/177033345-6bd970f2-44cd-410b-9780-23ab64edc45f.png)
从上面逻辑可以看到fcheck_files函数核心就是检查fd是否超出max_fds

 根据fd找到file之后，接下来就是从file读取数据。即ret = vfs_read(f.file, buf, count,ppos);
 ![image](https://user-images.githubusercontent.com/36918717/177033353-c7acfc51-16ee-4b22-8c3f-a330ee297c95.png)
![image](https://user-images.githubusercontent.com/36918717/177033354-e78c6f8d-6b25-42d6-a72a-53c1cbfd0066.png)

 ![image](https://user-images.githubusercontent.com/36918717/177033358-5cac1ae4-ac58-436d-b337-816f881205b6.png)

![image](https://user-images.githubusercontent.com/36918717/177033359-37ab9f39-d5ff-4828-8d05-f967fce4895c.png)
针对xfs，会调用fs/xfs/xfs_file.c里的xfs_file_read_iter()函数
![image](https://user-images.githubusercontent.com/36918717/177033362-74c82081-58ff-4fc3-9c8d-cc9394952a78.png)
![image](https://user-images.githubusercontent.com/36918717/177033363-2a9d0038-1ffb-40de-970d-ea46896bc268.png)
![image](https://user-images.githubusercontent.com/36918717/177033365-155002af-f35f-416b-8563-e9bf32a7e138.png)
 generic_file_read_iter()函数定义在mm/filemap.c文件中
![image](https://user-images.githubusercontent.com/36918717/177033369-266d7cd8-1bde-4c6e-ae6c-e3392a9cf3b0.png)
direct_IO()是个函数指针，针对不同文件系统会调用对应的函数，本文以xfs为例，但xfs没有实现direct_IO()逻辑，因此会调用fs/block_dev.c文件里的direct_IO()，即：
![image](https://user-images.githubusercontent.com/36918717/177033371-bda7d91f-cb33-4667-a32b-522411487d1d.png)
![image](https://user-images.githubusercontent.com/36918717/177033374-7fba5200-4482-48ff-a57f-6afe83c69ea9.png)


![image](https://user-images.githubusercontent.com/36918717/177033375-129510ae-0334-43d7-b086-abbd4ac2d793.png)


遍历获取缓冲区数据，并按照BIO_MAX_PAGES（256）个page数提交给devicedriver layer，即submit_bio()函数，接下来看下submit_bio()函数逻辑：
![image](https://user-images.githubusercontent.com/36918717/177033379-54695395-10d1-4a9f-bc8f-db62a1f36011.png)


generic_make_request()函数主要通过
![image](https://user-images.githubusercontent.com/36918717/177033384-169797b2-e0f8-493b-a526-96ef09a637e2.png)

generic_make_request()函数回调q->make_request_fn()函数，make_request_fn()在块设备初始化时调用blk_mq_init_queue()设置回调函数，函数定义在在block/blk-mq.c文件，在块设备初始化时设置给队列发送消息的回调函数。

blk_mq_init_queue()-> blk_mq_init_allocated_queue()->blk_queue_make_request()
![image](https://user-images.githubusercontent.com/36918717/177033391-4af02a3d-cbed-4170-a090-adcd8629ddff.png)
其中mfn就是blk_mq_make_request()。

接下来看下blk_mq_make_request()函数：
![image](https://user-images.githubusercontent.com/36918717/177033399-f35819a2-c659-4c63-875d-61b4ffbdace7.png)
blk_mq_get_request()函数核心部分是tag = blk_mq_get_tag(data);

blk_mq_get_tag()函数核心部分：
![image](https://user-images.githubusercontent.com/36918717/177033407-783feb7c-9eda-4b56-ac19-55d4e92c3bd3.png)
下图表示blk_mq_make_request()函数的核心调用逻辑

![image](https://user-images.githubusercontent.com/36918717/177033411-8922e1f7-eb0b-4245-b8b5-465ddb75f245.png)
deadline_next_request完成的功能本质上就是从hctx->queue->elevator->elevator_data->next_rq中取出一个元素。

  这儿从blk_mq_run_hw_queue函数出发，到最终的scsi_dispatch_cmd函数的调用，发现消息以scsi_cmnd的形式发送到了底层驱动。

 这些消息是从dd->next_rq[data_dir]中获取的，这儿的dd是deadline_data，但能够被发送出去之前，需要有地方入写到这里面来。







