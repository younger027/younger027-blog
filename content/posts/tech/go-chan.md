---
title: "Go Chan" #标题
date: 2023-02-20T15:05:49+08:00 #创建时间
lastmod: 2023-02-20T15:05:49+08:00 #更新时间
author: ["Younger"] #作者
categories:
- 分类1
- 分类2

tags:
- golang
- 源码

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

Do not communicate by sharing memory;instead, share memory by communicate.不要通过共享内存来通信，相反，应该通过通信来共享内存。这是Go语言并发的哲学座右铭。每个go开发在学习channel之前，应该先理解这个原则。



单纯地将函数并发执行是没有意义的。函数与函数间需要交换数据才能体现并发执行函数的意义。

虽然可以使用共享内存进行数据交换，但是共享内存在不同的goroutine中容易发生竞态问题。为了保证数据交换的正确性，必须使用互斥量对内存进行加锁，这种做法势必造成性能问题。

Go语言的并发模型是CSP（Communicating Sequential Processes），提倡通过通信共享内存而不是通过共享内存而实现通信。

如果说goroutine是Go程序并发的执行体，channel就是它们之间的连接。channel是可以让一个goroutine发送特定值到另一个goroutine的通信机制。

Go 语言中的通道（channel）是一种特殊的类型。通道像一个传送带或者队列，总是遵循先入先出（First In First Out）的规则，保证收发数据的顺序。每一个通道都是一个具体类型的导管，也就是声明channel的时候需要为其指定元素类型。(以上内容引自书籍[并发编程](https://www.topgoer.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/channel.html)对channel的解释)

#### 数据结构

#### 发送数据的过程

#### 接收数据的过程

#### 如何安全的关闭chan

#### Notification

1.mallocgc和malloc调用底层函数brk，bbrk做了什么

2.源码中的quick path如何理解，可以快速排除异常情况

3.sudoG是什么，双层缓存sudog是为了做什么

4.gopark。goready大概做了什么

5.如何安全的关闭chan

