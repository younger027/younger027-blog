---
title: "Mysql Interview"
date: 2023-08-02T19:07:39+08:00
draft: false
author: ["Younger"] #作者
categories:
- 分类1
- 分类2
tags:
- mysql
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

### 1. mysql b+树一般几层 如何计算？
- 为什么使用B+ Tree而不是B树？
  -  B Tree树的所有节点不仅存储索引也存数据，而B+ Tree只有叶子节点存数据，非叶子节点只存储索引，故一页B+ Tree比B Tree能存储更多的索引数据，树层高不会很高，这样我们查找数据的时候磁盘IO就更少，性能也就越好，因为系统和磁盘交互数据都是以页为单位的。
- 为什么mysql建议存储数据是2kw呢？如何计算
  - 我们以每条记录大小为1k，主键类型8byte，指针在mysql占6byte，一页为16k,那么非叶子节点就可以存储索引数据16*1024/(8+6)=1170条，那么二层的树就有1170 * 16k = 18720条记录，三层就有1170 * 1170 * 18720 = 21902400，2190w条记录，约2000w条记录。
  - 2kw也只是推荐值，超过了这个值可能会导致B+树层级更高，影响查询性能。


面试题，有100w滴滴司机，司机每3s上报一次自己的位置。请你设计一个系统，能够实时的将司机的位置存储进系统，并且能够给另外一个系统实时显示路径（每3秒刷新一次最新点），历史数据要能够随时可以查询
说说你的思路和实现？