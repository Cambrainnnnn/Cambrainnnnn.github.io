---
title: "Golang 调度器模式原理简析以及应用"
subtitle: ""
layout: post
author: "Quinlan"
header-style: text
hidden: true
tags:
  - golang
---

Golang 调度器模式原理简析以及应用
> 本文将简要分析 golang 调度器的基本原理，并重点阐述在日常开发中需要注意事项

# 协程基本概念
协程是一个计算机概念，goroutine 是 golang 对协程的实现。
```golang
type struct g {

}
```

进程是操作系统抽象的硬件资源的持有者。这里的硬件资源，包括网络端口，文件描述符，内存等。线程能够使用进程的资源，其是多任务操作系统中竞争 CPU 的最小单位。协程是减少 CPU 在线程间上下文切换，来保证更高效利用 CPU 资源的函数调度实体。
所以进程虽然有很多线程，但某个时刻只有一个线程能去访问互斥资源。同理线程中虽然有很多协程，但某个时刻只有一个协程可以使用 CPU 资源。



# 调度器常见算法
调度器用于管理协程状态以及对线程多路复用；


# 日常开发



**参考资料**
1. [kotlin coroutine](https://subscription.packtpub.com/book/programming/9781788627160/1/ch01lvl1sec02/processes-threads-and-coroutines)
2. [goroutines under the hood](https://vitalcs.substack.com/p/goroutines-under-the-hood)



**一个小的疑问，Go 仅分配 CPU core 数量的线程，在容器部署的情况下，是否可能竞争能力更弱，导致更容易受到影响？**