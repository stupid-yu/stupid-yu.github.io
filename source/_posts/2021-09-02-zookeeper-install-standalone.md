---
title: ZooKeeper 本地模式安装
date: 2021-09-02 14:37:59
tags: zookeeper
categories: 分布式
---
# ZooKeeper 本地模式

> Running ZooKeeper in standalone mode is convenient for evaluation, some development, and testing.

## 先决条件
> See [System Requirements](https://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_systemReq) in the Admin guide.

+ ZooKeeper runs in Java, release 1.8 or greater (JDK 8 LTS, JDK 11 LTS, JDK 12 - Java 9 and 10 are not supported).

## Download

[ZooKeeper-3.5.6](http://archive.apache.org/dist/zookeeper/zookeeper-3.5.6/apache-zookeeper-3.5.6-bin.tar.gz)

![](https://cdn.jsdelivr.net/gh/stupid-yu/cdn/img/zookeeper-3.5.6.png)

<!--more-->

## Standalone Operation

### 配置修改

```bash
# 解压到指定目录 （root用户）
[root@archtx Downloads]# tar -zxvf apache-zookeeper-3.5.6-bin.tar.gz -C /opt/

# 修改目录名称
[root@archtx opt]# mv apache-zookeeper-3.5.6-bin apache-zookeeper-3.5.6

# 配置修改 conf目录下zoo_sample.cfg => zoo.cfg
[root@archtx conf]# mv zoo_sample.cfg zoo.cfg
```

### 操作 ZooKeeper

```bash
# 启动 ZooKeeper
[root@archtx apache-zookeeper-3.5.6]# bin/zkServer.sh start
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/apache-zookeeper-3.5.6/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

# 查看进程是否启动
[root@archtx apache-zookeeper-3.5.6]# jps
57045 Jps
56919 QuorumPeerMain

# 查看状态
[root@archtx apache-zookeeper-3.5.6]# bin/zkServer.sh status
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/apache-zookeeper-3.5.6/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: standalone

# 启动客户端
[root@archtx apache-zookeeper-3.5.6]# bin/zkCli.sh 
/usr/bin/java
Connecting to localhost:2181
.....
Welcome to ZooKeeper!
2021-09-02 16:03:25,244 [myid:localhost:2181] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1112] - Opening socket connection to server localhost/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2021-09-02 16:03:25,317 [myid:localhost:2181] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@959] - Socket connection established, initiating session, client: /127.0.0.1:54148, server: localhost/127.0.0.1:2181
2021-09-02 16:03:25,331 [myid:localhost:2181] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1394] - Session establishment complete on server localhost/127.0.0.1:2181, sessionid = 0x1000a8aa8090000, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: localhost:2181(CONNECTED) 0] 

# 退出客户端
[zk: localhost:2181(CONNECTED) 0] quit
WATCHER::

WatchedEvent state:Closed type:None path:null
2021-09-02 16:05:57,776 [myid:] - INFO  [main-EventThread:ClientCnxn$EventThread@524] - EventThread shut down for session: 0x1000a8aa8090000
2021-09-02 16:05:57,776 [myid:] - INFO  [main:ZooKeeper@1422] - Session: 0x1000a8aa8090000 closed

# 停止ZooKeeper
[root@archtx apache-zookeeper-3.5.6]# bin/zkServer.sh stop
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/apache-zookeeper-3.5.6/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
```

## 配置参数解读
```
-------------------zoo_sample.cfg content begin------------------
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/tmp/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
```
>+ tickTime : the basic time unit in milliseconds used by ZooKeeper. It is used to do heartbeats and the minimum session timeout will be twice the tickTime.
>
>+ dataDir : the location to store the in-memory database snapshots and, unless specified otherwise, the transaction log of updates to the database.
>
>+ clientPort : the port to listen for client connections

| 参数        | 默认值        |含义                                   |
|:------------|:--------------|:--------------------------------------|
|tickTime     | 2000毫秒      |ZooKeeper 服务器和客户端通信心跳时间   |
|initLimit    | 10 心跳数     |LF初始通信时限制(LF:Leader-Follower)   |
|syncLimit    | 5 心跳数      |LF同步通信时限                         |
|dataDir      | /tmp/zookeeper|保存ZooKeeper中的数据，一般不用默认    |
|clientPort   | 2181          | 客户端连接端口，通常不做修改          |





