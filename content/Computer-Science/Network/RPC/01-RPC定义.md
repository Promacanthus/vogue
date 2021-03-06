---
title: "01 RPC定义"
date: 2020-05-14T14:12:30+08:00
draft: true
---

- [0.1. RPC定义](#01-rpc定义)
  - [0.1.1. RPC通信流程](#011-rpc通信流程)
  - [0.1.2. RPC在架构中的位置](#012-rpc在架构中的位置)

## 0.1. RPC定义

RPC 的全称是 Remote Procedure Call，即远程过程调用。它是帮助我们屏蔽网络编程细节，实现调用远程方法就跟调用本地（同一个项目中的方法）一样的体验，我们不需要因为这个方法是远程调用就需要编写很多与业务无关的代码。

RPC 的作用：

- **屏蔽**远程调用跟本地调用的区别，让我们感觉就是调用项目内的方法
- **隐藏**底层网络通信的复杂性，让我们更专注于业务逻辑

RPC 最大的特点就是可以让我们像调用本地一样发起远程调用，这一特点让人感觉 RPC 就是为“微服务”或 SOA 而生的。

RPC 不是只应用在“微服务”中，只要涉及到网络通信，我们就可能用到 RPC，它是解决分布式系统**通信问题**的一大利器，它对网络通信的整个过程做了完整封装，在搭建分布式系统时，使网络通信逻辑的开发变得更加简单，同时也会让网络通信变得更加安全可靠。

> 分布式系统中的网络通信一般都会采用四层的 `TCP` 协议或七层的 `HTTP` 协议，其中，前者占大多数，这主要得益于 `TCP` 协议的稳定性和高效性。网络通信是一个复杂的过程，主要包括：1-对端节点的查找、2-网络连接的建立、3-传输数据的编码解码、4-网络连接的管理等。

使用 RPC 可以像调用本地一样发起远程调用，用它解决通信问题：

- 序列化
- 编解码
- 网络传输

这些是 RPC 的基础，它真正强大的地方是治理功能：

- 连接管理
- 健康检测
- 负载均衡
- 优雅启停
- 异常重试
- 业务分组
- 熔断限流

### 0.1.1. RPC通信流程

![image](/images/acf53138659f4982bbef02acdd30f1fa.jpg)

### 0.1.2. RPC在架构中的位置

RPC 是解决应用间通信的一种方式，应用架构最终都会从“单体”演进成“微服务化”，RPC 框架帮助我们解决系统拆分后的通信问题，让我们像调用本地一样去调用远程方法，可以说 RPC 对应的是整个分布式应用系统，就像是“经络”一样的存在。

![image](/images/506e902e06e91663334672c29bfbc2be.jpg)

在这个应用中，使用了 MQ 来处理异步流程、Redis 缓存热点数据、MySQL 持久化数据，还有就是在系统中调用另外一个业务系统的接口，对应用来说这些都是属于 RPC 调用，而 MQ、MySQL 持久化的数据也会存在于一个分布式文件系统中，他们之间的调用也是需要用 RPC 来完成数据交互的。

> RPC 是日常开发中经常接触的东西，只是被包装成了各种框架，导致我们很少意识到这就是 RPC。
