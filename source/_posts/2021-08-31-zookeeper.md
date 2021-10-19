---
title: ZooKeeper 简介
date: 2021-08-31 15:15:58
tags: zookeeper
categories: 分布式
---
# ZooKeeper

> A Distributed Coordination Service for Distributed Applications

## 概述

> ZooKeeper: Because Coordinating Distributed Systems is a Zoo

[ZooKeeper](https://zookeeper.apache.org/) 是分布式应用的分布式开源协调服务。

<!--more-->

### 工作机制

ZooKeeper 从设计模式的角度来理解：是一个基于观察者模式设计的分布式管理框架，它***负责存储和管理大家都关心的数据***，然后***接受观察者的注册***，一旦这些数据发生变化，ZooKeeper 就将***负责通知已经在ZooKeeper上注册的那些观察者***作出相应的反应。

## 特点

![](https://zookeeper.apache.org/doc/current/images/zkservice.jpg)

> Guarantees
ZooKeeper is very fast and very simple. Since its goal, though, is to be a basis for the construction of more complicated services, such as synchronization, it provides a set of guarantees. These are:
> 
>+ Sequential Consistency - Updates from a client will be applied in the order that they were sent.
>
>+ Atomicity - Updates either succeed or fail. No partial results.
>
>+ Single System Image - A client will see the same view of the service regardless of the server that it connects to. i.e., a client will never see an older view of the system even if the client fails over to a different server with the same session.
> 
>+ Reliability - Once an update has been applied, it will persist from that time forward until a client overwrites the update.
>
>+ Timeliness - The clients view of the system is guaranteed to be up-to-date within a certain time bound.

ZooKeeper： 一个 Leader,多个 Follower 组成的集群，集群中只要有半数以上节点存活，ZooKeeper 集群就能正常服务。所以 ZooKeeper 适合安装奇数台服务器。其特点总结如下：
+ 更新请求顺序执行，来自同一个 Client 的请求按其发送顺序依次执行。
+ 全局数据一致性：每个 Server 保存一份相同的数据副本，Client 无论连接到哪个 Server，数据都是一致的。
+ 数据更新原子性，一次数据更新要么成功，要么失败。
+ 可靠性，数据保存直到下次 Client 覆盖更新。
+ 实时性，在一定时间范围内，Client 能读到最新的数据。

## Data model and the hierarchical namespace

> The namespace provided by ZooKeeper is much like that of a standard file system. A name is a sequence of path elements separated by a slash (/). Every node in ZooKeeper's namespace is identified by a path.

![](https://zookeeper.apache.org/doc/current/images/zknamespace.jpg)

ZooKeeper 数据结构与 Unix 文件系统类似，整体上可以看作一棵树，每个节点称作一个 ZNode。每一个 ZNode 默认能够存储 ***1MB*** 的数据，每个 ZNode 都可以通过***其路径唯一标识***。

## Simple API

> One of the design goals of ZooKeeper is providing a very simple programming interface. As a result, it supports only these operations:

+ create : creates a node at a location in the tree
+ delete : deletes a node
+ exists : tests if a node exists at a location
+ get data : reads the data from a node
+ set data : writes data to a node
+ get children : retrieves a list of children of a node
+ sync : waits for data to be propagated 

## 应用场景

+ 统一命名服务
+ 统一配置管理
+ 统一集群管理
+ 服务节点动态上下线
+ 软负载均衡

## 总结

简言之，ZooKeeper = 文件系统 + 通知机制。