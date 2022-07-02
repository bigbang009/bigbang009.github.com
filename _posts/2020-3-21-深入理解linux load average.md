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


