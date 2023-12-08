---
title: "Online Case DiskIO"
date: 2023-08-18T17:41:26+08:00
author: ["Younger"] #作者
categories:
- 分类1
- 分类2
tags:
- online case
- interview
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
image: "" #图片路径：posts/tech/文章1/picture.png
caption: "" #图片底部描述
alt: ""
relative: false
---


线上接口超时告警，发现接口响应时长变长。但是cpu和内存没有明显的波动。
打开pprof监控，发现thread的数量达到1k。gpm模型下，os的thread一般不会太大，一些高频的服务，也不会超过200.
那么是什么导致创建这么多线程呢。gpm的模型中的sysmon协程，它会轮询的检测所有的P上下文队列，只要 G-M 的线程长时间在阻塞状态，那么就重新创建一个线程去从runtime P队列里获取任务。先前的阻塞的线程会被游离出去了，当他完成阻塞操作后会触发相关的callback回调，并加入回线程组里。简单说，如果你没有特意配置runtime.SetMaxThreads，那么在可没有复用的线程下会一直创建新线程。 golang默认的最大线程数10000个线程，这个是硬编码。 如果想要控制golang的pthread线程数可以使用 runtime.SetMaxThreads() 。

那么到底是什么操作导致创建了这么多的thread呢，G为什么会阻塞呢。然后继续分析pprof的stack dump会发现有很多的log调用。然后就去机器上查看iostat查看磁盘的io情况，发现wsec/s数值比较高，而且%utils是100。由此可定位是log写的太多，导致磁盘io到达瓶颈，log太多，又会导致过多的M被创建，过多的M又会引起上下文切换，过多的调度开销。也会引起cpu cache miss。

解决方案就是设置最大线程数200，然后将log从debug调级到warn。

log的写入流程是，先将内容写到os内核的page cache中，然后操作系统会通过阈值或定时刷新，将数据落到磁盘上。大量的日志写入，都会先扔到page cache里，导致os需要频繁的flush数据到磁盘。磁盘io压力很大，到达瓶颈后不能很快flush，开始阻塞，级联的导致golang写日志G阻塞，也就导致M的增加。所以线上的问题都是层层关联的。你需要抽丝剥茧的找出异常的点，然后还得通盘的想清楚级联的逻辑，也就是为什么会互相影响。比如从写log，到log落盘机制，然后还得理解go的gmp，才能串起来整个问题链路。这就需要丰富的知识积累了。
### 现象

### 排查方法
### 定位原因
### 解决方案
### 总结