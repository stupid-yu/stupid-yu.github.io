---
title: Kafka 授权机制
date: 2021-11-29 17:28:25
tags: kafka
categories: 
  - [分布式]
  - [Messaging System, Kafka]
---
## 前言

所谓授权，一般是指对与信息安全或计算机安全相关的资源授予访问权限，特别是**存取**控制。

具体到权限模型，常见的有四种。

+ ACL：Access-Control List，访问控制列表。权限模型
+ RBAC：Role-Based Access Control，基于角色的权限控制。
+ ABAC：Attribute-Based Access Control，基于属性的权限控制。
+ PBAC：Policy-Based Access Control，基于策略的权限控制。

权限模型在典型的互联网场景中，前两种模型应用得多，后面这两种则比较少用。

<!--more-->

## Kafka 授权机制

Kafka 没有使用 RBAC 模型，它用的是 ACL 模型。简单来说，这种模型就是规定了什么用户对什么资源有什么样的访问权限。

> **Principal P is [Allowed/Denied] Operation O From Host H On Resource R**
>
> + Principal：表示访问 Kafka 集群的用户。
> + Operation：表示一个具体的访问类型，如读写消息或创建主题等。
> + Host：表示连接 Kafka 集群的客户端应用程序 IP 地址。Host 支持星号占位符，表示所有 IP 地址。
> + Resource：表示 Kafka 资源类型。如果以2.3 版本为例，Resource 共有 5 种，分别是 TOPIC、CLUSTER、GROUP、TRANSACTIONALID 和 DELEGATION TOKEN。

当前，Kafka 提供了一个可插拔的授权实现机制。该机制会将你配置的所有 ACL 项保存在 ZooKeeper 下的 /kafka-acl 节点中。你可以通过 Kafka 自带的 kafka-acls 脚本动态地对 ACL 项进行增删改查，并让它立即生效。

## Kafka ACL常用操作

### 配置 Broker 端 `server.properties`

```properties
# authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
# 设置ACL类(高于 2.4.0 版本推荐使用 AclAuthorizer)
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```
### 超级用户（Super User）

在开启了 ACL 授权之后，你还必须显式地为不同用户设置访问某项资源的权限，否则，在默认情况下，没有配置任何 ACL 的资源是不能被访问的。不过，这里也有一个例外：超级用户能够访问所有的资源，即使你没有为它们设置任何 ACL 项。

在一个 Kafka 集群中设置超级用户，只需要在 Broker 端的配置文件 `server.properties` 中，设置 `super.users` 参数即可，比如：

```properties
# 分号分割
super.users=User:superuser1;User:superuser2
```

除了设置 `super.users `参数，Kafka 还支持将所有用户都配置成超级用户的用法。如果我们在 `server.properties` 文件中设置 `allow.everyone.if.no.acl.found=true`，那么所有用户都可以访问没有设置任何 ACL 的资源。

不过不太建议进行这样的设置。毕竟，在生产环境中，特别是在那些对安全有较高要求的环境中，采用白名单机制要比黑名单机制更加令人放心。

### kafka-acls 脚本

```bash
# 为用户 Alice 增加了集群级别的所有权限
[dev@hp102 kafka-2.4.1]$ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Alice --operation All --topic '*' --cluster

# 允许所有的用户使用任意的 IP 地址读取名为 test-topic 的主题数据，同时也禁止 BadUser 用户和 10.205.96.119 的 IP 地址访问 test-topic 下的消息。
[dev@hp102 kafka-2.4.1]$ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:'*' --allow-host '*' --deny-principal User:BadUser --deny-host 10.205.96.119 --operation Read --topic test-topic
```

### 授权机制能否单独使用

关于授权，有一个很常见的问题是，Kafka 授权机制能不配置认证机制而单独使用吗？其实，这是可以的，只是你只能为 IP 地址设置权限。比如，下面这个命令会禁止运行在 127.0.0.1 IP 地址上的 Producer 应用向 test 主题发送数据：

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --deny-principal User:* --deny-host 127.0.0.1 --operation Write --topic test
```

##  SSL + ACL 配置的实例

### SSL 配置

> SSL 配置参考 kafka 认证机制 ssl部分。

### ACL 配置

#### 开启 ACL

配置 Broker 端 `server.properties`

```bashauthorizer.class.name=kafka.security.auth.SimpleAclAuthorizer。```

#### 白名单机制

建议采用白名单机制，这样的话，没有显式设置权限的用户就无权访问任何资源。

####  授予 SSL 用户集群的权限

使用 kafka-acls 脚本为 SSL 用户授予集群的权限，我们以前面的例子来进行一下说明。在配置 SSL 时，我们指定用户的 Distinguished Name 为 "CN=Hui Yu, OU=Bcdp, O=Bonc, L=Beijing, ST=Beijing, C=CN"。之前在设置 Broker 端参数时，我们指定了 `security.inter.broker.protocol=SSL`，即强制指定 Broker 间的通讯也采用 SSL 加密。如果不为指定的 Distinguished Name 授予集群操作的权限，你是无法成功启动 Broker 的。因此，你需要在启动 Broker 之前执行下面的命令：

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:"CN=Hui Yu,OU=Bcdp,O=Bonc,L=Beijing,ST=Beijing,C=CN" --operation All --cluster
```

#### 授予客户端程序相应的权限

你要为客户端程序授予相应的权限，比如为生产者授予 producer 权限，为消费者授予 consumer 权限。假设客户端要访问的主题名字是 test，那么命令如下：

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:"CN=Hui Yu,OU=Bcdp,O=Bonc,L=Beijing,ST=Beijing,C=CN" --producer --topic test

[dev@hp102 kafka-2.4.1]$ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:"CN=Hui Yu,OU=Bcdp,O=Bonc,L=Beijing,ST=Beijing,C=CN" --consumer --topic test
```

## 总结

> 最好不要把其他权限授予客户端，比如创建主题的权限。

总之，你授予的权限越少，你的 Kafka 集群就越安全。

## 思考题

Q: 如果要让一个客户端能够查询消费者组的提交位移数据，你觉得应该授予它什么权限？

A: 消费者端的TOPIC的WRITE权限 [源码规定]

附:
![img](https://cdn.jsdelivr.net/gh/stupid-yu/CDN/img/kafka-acl.jpg)