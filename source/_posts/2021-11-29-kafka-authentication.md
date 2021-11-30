---
title: Kafka 认证机制
date: 2021-11-29 17:15:38
tags: kafka
categories: 
  - [分布式]
  - [Messaging System, Kafka]
---

## 前言

所谓认证，又称“验证”“鉴权”，英文是 authentication，是指通过一定的手段，完成对用户身份的确认。认证的主要目的是确认当前声称为某种身份的用户确实是所声称的用户。

> 在计算机领域，经常和认证搞混的一个术语就是授权，英文是 authorization。授权一般是指对信息安全或计算机安全相关的资源定义与授予相应的访问权限。举个简单的例子来区分下两者：认证要解决的是你要证明你是谁的问题，授权要解决的则是你能做什么的问题。
>
> **在 Kafka 中，认证和授权是两套独立的安全配置。**

<!--more-->

## Kafka 认证机制

自 0.9.0.0 版本开始，Kafka 正式引入了认证机制，用于实现基础的安全用户认证。Kafka 支持基于 SSL 和基于 SASL 的安全认证机制。

> 目前来看，使用 SSL 做信道加密的情况更多一些，但使用 SSL 实现认证不如使用 SASL。
>
> 可以使用 SSL 来做通信加密，使用 SASL 来做 Kafka 的认证实现

+ 基于 SSL 的认证主要是指 Broker 和客户端的双路认证（2-way authentication）。
+ Kafka 还支持通过 SASL 做客户端认证。

> Kafka supports the following SASL mechanisms:
>
> - [GSSAPI](https://kafka.apache.org/24/documentation/#security_sasl_kerberos) (Kerberos) (v0.9.0.0)
> - [PLAIN](https://kafka.apache.org/24/documentation/#security_sasl_plain) (v0.10.0.0)
> - [SCRAM-SHA-256](https://kafka.apache.org/24/documentation/#security_sasl_scram) (v0.10.2.0)
> - [SCRAM-SHA-512](https://kafka.apache.org/24/documentation/#security_sasl_scram) (v0.10.2.0)
> - [OAUTHBEARER](https://kafka.apache.org/24/documentation/#security_sasl_oauthbearer) (v2.0)

上面展示的四种是kafka官方推荐的SASL认证方式。

| 认证方式         |                             说明                             |
| :--------------- | :----------------------------------------------------------: |
| SASL/GSSAPI      | 主要是给 Kerberos 用户使用的，如果当前已经有了Kerberos认证，只需要给集群中每个Broker和访问用户申请Principals，然后在Kafka的配置文件中开启Kerberos的支持即可，官方参考：[Authentication using SASL/Kerberos]。 |
| SASL/PLAIN       | 是一种简单的用户名/密码身份验证机制，通常与TLS/SSL一起用于加密，以实现安全身份验证。是一种比较容易使用的方式，但是有一个很明显的缺点，这种方式会把用户账户文件配置到一个静态文件中，每次想要添加新的账户都***需要重启Kafka***去加载静态文件，才能使之生效，十分的不方便，官方参考[Authentication using SASL/PLAIN]。 |
| SASL/SCRAM       | 通过将认证用户信息保存在 ZooKeeper 里面，从而动态的获取用户信息，相当于把ZK作为一个认证中心使用了。这种认证可以在使用过程中，使用 Kafka 提供的命令动态地创建和删除用户，无需重启整个集群，十分方便。官方参考[Authentication using SASL/SCRAM]。 |
| SASL/OAUTHBEARER | kafka 2.0 版本引入的新认证机制，主要是为了实现与 OAuth 2 框架的集成。Kafka 不提倡单纯使用 OAUTHBEARER，因为它生成的不安全的 Json Web Token，必须配以 SSL 加密才能用在生产环境中。官方参考[Authentication using SASL/OAUTHBEARER] 。 |

### 认证机制总结

![](https://cdn.jsdelivr.net/gh/stupid-yu/CDN/img/kafka-authentication-mechanisms.jpg)



## SASL/SCRAM-SHA-256 配置实例

### 第 1 步：创建用户

配置 SASL/SCRAM 的第一步，是创建能否连接 Kafka 集群的用户。

在本次测试中，创建 3 个用户，分别是 admin 用户、writer 用户和 reader 用户。admin 用户用于实现 Broker 间通信，writer 用户用于生产消息，reader 用户用于消费消息。

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-configs.sh --zookeeper hp102:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin],SCRAM-SHA-512=[password=admin]' --entity-type users --entity-name admin

[dev@hp102 kafka-2.4.1]$ bin/kafka-configs.sh --zookeeper hp102:2181 --alter --add-config 'SCRAM-SHA-256=[password=writer],SCRAM-SHA-512=[password=writer]' --entity-type users --entity-name writer

[dev@hp102 kafka-2.4.1]$ bin/kafka-configs.sh --zookeeper hp102:2181 --alter --add-config 'SCRAM-SHA-256=[password=reader],SCRAM-SHA-512=[password=reader]' --entity-type users --entity-name reader
```

### 第 2 步：创建 JAAS 文件

`kafka_server_jass.conf`

```properties
KafkaServer {
org.apache.kafka.common.security.scram.ScramLoginModule required
username="admin"
password="admin";
};
```

### 第 3 步：配置 `server.properties`

```properties
listeners=SASL_PLAINTEXT://hp102:9092

security.inter.broker.protocol=SASL_PLAINTEXT

sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256

sasl.enabled.mechanisms=SCRAM-SHA-256
```

### 第 4 步：启动Broker

```bash
[dev@hp102 kafka-2.4.1]$ KAFKA_OPTS=-Djava.security.auth.login.config=config/kafka_jaas.conf bin/kafka-server-start.sh -daemon config/server.properties
```

### 第 5 步：创建Topic

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-topic.sh --zookeeper hp102:2181 --create --topic topic-demo --partitions 2 --replication-factor 3
```

### 第 6 步：发送消息

`producer.conf`的配置文件

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="writer" password="writer";
```

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-console-producer.sh --broker-list hp102:9092 --topic topic-demo --producer.config config/producer.conf
```

### 第 7 步：消费消息

`consumer.conf`的配置文件

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="reader" password="reader";
```

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-console-consumer.sh --bootstrap-server hp102:9092 --topic topic-demo --consumer.config config/consumer.conf
```

## SSL 配置实例

### 第1步：利用提供的`kafka-generate-ssl.sh`脚本

> 或者选择官方提供的脚本[`kafka-generate-ssl.sh`](https://raw.githubusercontent.com/confluentinc/confluent-platform-security-tools/master/kafka-generate-ssl.sh)

```bash
#!/bin/bash

#设置环境变量
BASE_DIR=/home/dev/testenv #你需要修改此处
CERT_OUTPUT_PATH="$BASE_DIR/certificates"
PASSWORD=test1234
KEY_STORE="$CERT_OUTPUT_PATH/server.keystore.jks"
TRUST_STORE="$CERT_OUTPUT_PATH/server.truststore.jks"
CLIENT_KEY_STORE="$CERT_OUTPUT_PATH/client.keystore.jks"
CLIENT_TRUST_STORE="$CERT_OUTPUT_PATH/client.truststore.jks"
KEY_PASSWORD=$PASSWORD
STORE_PASSWORD=$PASSWORD
TRUST_KEY_PASSWORD=$PASSWORD
TRUST_STORE_PASSWORD=$PASSWORD
CERT_AUTH_FILE="$CERT_OUTPUT_PATH/ca-cert"
DAYS_VALID=365
DNAME="CN=Hui Yu, OU=Bcdp, O=Bonc, L=Beijing, ST=Beijing, C=CN"


mkdir -p $CERT_OUTPUT_PATH

echo "1. 产生key和证书......"
keytool -keystore $KEY_STORE -alias kafka-server -validity $DAYS_VALID -genkey -keyalg RSA \
-storepass $STORE_PASSWORD -keypass $KEY_PASSWORD -dname "$DNAME"

keytool -keystore $CLIENT_KEY_STORE -alias kafka-client -validity $DAYS_VALID -genkey -keyalg RSA \
-storepass $STORE_PASSWORD -keypass $KEY_PASSWORD -dname "$DNAME"

echo "2. 创建CA......"
openssl req -new -x509 -keyout $CERT_OUTPUT_PATH/ca-key -out "$CERT_AUTH_FILE" -days "$DAYS_VALID" \
-passin pass:"$PASSWORD" -passout pass:"$PASSWORD" \
-subj "/C=CN/ST=Beijing/L=Beijing/O=Bonc/OU=Bcdp,CN=Hui Yu"

echo "3. 添加CA文件到broker truststore......"
keytool -keystore "$TRUST_STORE" -alias CARoot \
-importcert -file "$CERT_AUTH_FILE" -storepass "$TRUST_STORE_PASSWORD" -keypass "$TRUST_KEY_PASS" -noprompt

echo "4. 添加CA文件到client truststore......"
keytool -keystore "$CLIENT_TRUST_STORE" -alias CARoot \
-importcert -file "$CERT_AUTH_FILE" -storepass "$TRUST_STORE_PASSWORD" -keypass "$TRUST_KEY_PASS" -noprompt

echo "5. 从keystore中导出集群证书......"
keytool -keystore "$KEY_STORE" -alias kafka-server -certreq -file "$CERT_OUTPUT_PATH/server-cert-file" \
-storepass "$STORE_PASSWORD" -keypass "$KEY_PASSWORD" -noprompt

keytool -keystore "$CLIENT_KEY_STORE" -alias kafka-client -certreq -file "$CERT_OUTPUT_PATH/client-cert-file" \
-storepass "$STORE_PASSWORD" -keypass "$KEY_PASSWORD" -noprompt

echo "6. 使用CA签发证书......"
openssl x509 -req -CA "$CERT_AUTH_FILE" -CAkey $CERT_OUTPUT_PATH/ca-key -in "$CERT_OUTPUT_PATH/server-cert-file" \
-out "$CERT_OUTPUT_PATH/server-cert-signed" -days "$DAYS_VALID" -CAcreateserial -passin pass:"$PASSWORD"

openssl x509 -req -CA "$CERT_AUTH_FILE" -CAkey $CERT_OUTPUT_PATH/ca-key -in "$CERT_OUTPUT_PATH/client-cert-file" \
-out "$CERT_OUTPUT_PATH/client-cert-signed" -days "$DAYS_VALID" -CAcreateserial -passin pass:"$PASSWORD"

echo "7. 导入CA文件到keystore......"
keytool -keystore "$KEY_STORE" -alias CARoot -import -file "$CERT_AUTH_FILE" -storepass "$STORE_PASSWORD" \
 -keypass "$KEY_PASSWORD" -noprompt

keytool -keystore "$CLIENT_KEY_STORE" -alias CARoot -import -file "$CERT_AUTH_FILE" -storepass "$STORE_PASSWORD" \
 -keypass "$KEY_PASSWORD" -noprompt

echo "8. 导入已签发证书到keystore......"
keytool -keystore "$KEY_STORE" -alias kafka-server -import -file "$CERT_OUTPUT_PATH/server-cert-signed" \
 -storepass "$STORE_PASSWORD" -keypass "$KEY_PASSWORD" -noprompt

keytool -keystore "$CLIENT_KEY_STORE" -alias kafka-client -import -file "$CERT_OUTPUT_PATH/client-cert-signed" \
 -storepass "$STORE_PASSWORD" -keypass "$KEY_PASSWORD" -noprompt

echo "9. 删除临时文件......"
rm "$CERT_OUTPUT_PATH/ca-cert.srl"
rm "$CERT_OUTPUT_PATH/server-cert-signed"
rm "$CERT_OUTPUT_PATH/client-cert-signed"
rm "$CERT_OUTPUT_PATH/server-cert-file"
rm "$CERT_OUTPUT_PATH/client-cert-file"
```

该脚本主要的产出是 4 个文件，分别是：server.keystore.jks、server.truststore.jks、client.keystore.jks 和 client.truststore.jks。

你需要把以 server 开头的两个文件，拷贝到集群中的所有 Broker 机器上，把以 client 开头的两个文件，拷贝到所有要连接 Kafka 集群的客户端应用程序机器上。

### 第 2 步：配置`server.properties`

```properties
listeners=SSL://hp102:9093
ssl.truststore.location=/home/dev/testenv/certificates/server.truststore.jks
ssl.truststore.password=test1234
ssl.keystore.location=/home/dev/testenv/certificates/server.keystore.jks
ssl.keystore.password=test1234
security.inter.broker.protocol=SSL
ssl.client.auth=required
ssl.key.password=test1234

# 若提示ssl handshake failed 配置
# 因为自 Kafka 2.0 版本开始，它默认会验证服务器端的主机名是否匹配 Broker 端证书里的主机名
ssl.endpoint.identification.algorithm=
```

### 第 3 步：启动Broker

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-server-start.sh -daemon config/server.properties
```

### 第 4 步：创建Topic

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-topic.sh --zookeeper hp102:2181 --create --topic test --partitions 2 --replication-factor 3
```

### 第 5 步：配置客户端的 SSL 

`client-ssl.config`

```properties
security.protocol=SSL
ssl.truststore.location=/home/dev/testenv/certificates/client.truststore.jks
ssl.truststore.password=test1234
ssl.keystore.location=/home/dev/testenv/certificates/server.keystore.jks
ssl.keystore.password=test1234
ssl.key.password=test1234
ssl.endpoint.identification.algorithm=
```

### 第 6 步：发送消息

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-console-producer.sh --broker-list hp102:9093 --topic test --producer.config config/client-ssl.config
```

### 第 7 步：消费消息

```bash
[dev@hp102 kafka-2.4.1]$ bin/kafka-console-consumer.sh --bootstrap-server hp102:9093 --topic test --consumer.config config/client-ssl.config
```

## 附

### SSL 证书生成

Kafka 允许 client 通过 SSL 连接。SSL默认情况下被禁止，但可以根据需要开启。

可以使用Java的keytool工具来完成，Keytool 是一个Java 数据证书的管理工具 ,Keytool 将密钥（key）和证书（certificates）存在一个称为keystore的文件中，在keystore里，包含两种数据： 

  1). 密钥实体（Key entity）——密钥（secret key）又或者是私钥和配对公钥（采用非对称加密） 

  2). 可信任的证书实体（trusted certificate entries）——只包含公钥

keytool相关指令说明：

| 名称       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| -alias     | 别名，可自定义，这里叫kafkaserver                            |
| -keystore  | 指定密钥库的名称(就像数据库一样的证书库，可以有很多个证书，ca certs这个文件是jre自带的， 也可以使用其它文件名字，如果没有这个文件名字，它会创建这样一个) |
| -storepass | 指定密钥库的密码                                             |
| -keypass   | 指定别名条目的密码                                           |
| -list      | 显示密钥库中的证书信息                                       |
| -export    | 将别名指定的证书导出到文件                                   |
| -file      | 参数指定导出到文件的文件名                                   |
| -import    | 将已签名数字证书导入密钥库                                   |
| -keypasswd | 修改密钥库中指定条目口令                                     |
| -dname     | 指定证书拥有者信息。其中，CN=主机名/域名,OU=组织单位名称,O=组织名称,L=城市或区域名称,ST=州或省份名称,C=单位的两字母国家代码 |
| -keyalg    | 指定密钥的算法                                               |
| -validity  | 指定创建的证书有效期多少天                                   |
| -keysize   | 指定密钥长度                                                 |

Kafka集群的每个broker节点生成SSL密钥和证书

> 集群中的每一台机器都有一个公私密钥对、一个标识该机器的证书

```bash
keytool -keystore server.keystore.jks -alias kafka-server -validity 365 -genkey
```

  执行命令时，输入first and last name，这里需要输入你的**主机名**，确保公用名（CN）与服务器的完全限定域名（FQDN）精确相匹配。

  client 拿 CN 与 DNS 域名进行比较以确保它确实连接到所需的服务器，而不是恶意的服务器。

### 生成CA认证证书

> 为了保证整个证书的安全性，认证机构（CA）负责签发证书。
>
> 认证机构就像发行护照的政府，政府会对每张护照盖章，使得护照很难被伪造。其它，政府核实印章，以保证此护照是真实的。
>
> 类似的，CA签发证书，密码保证签署的证书在计算上很难被伪造。因此，只要CA是一个真正值得信赖的权威机构，客户就可以很高的保证他们正在连接到真实的机器。

```bash
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
```

+ ***ca-cert*** : the certificate of the CA [CA证书]
+ ***ca-key***: the private key of the CA [CA私钥]

### 通过CA证书创建一个服务器端信任证书

> 有了信任证书才可以进行证书有效性的检测

```bash
keytool -keystore server.truststore.jks -alias CARoot -import -file ca-cert
```

### 通过CA证书创建一个客户端端信任证书

```bash
keytool -keystore client.truststore.jks -alias CARoot -import -file ca-cert
```

### 从密钥库导出证书服务器端证书cert-file

```bash
keytool -keystore server.keystore.jks -alias kafkaserver -certreq -file cert-file
```

### 通过CA证书给服务器端证书进行签名处理

```bash
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:{ca-password}
```

### 将CA证书导入到服务器端keystore

```bash
keytool -keystore server.keystore.jks -alias CARoot -import -file ca-cert
```

### 将已签名的服务器证书导入到服务器keystore

```bash
keytool -keystore server.keystore.jks -alias kafkaserver -import -file cert-signed
```

