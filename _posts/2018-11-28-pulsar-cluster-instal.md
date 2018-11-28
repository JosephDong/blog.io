---
layout: post
title: Pulsar 集群安装
date: 2018-11-28
categories: Pulsar
tags: [MQ,Pulsar,消息队列]
description: Pulsar 集群安装

---

### 0x1 摘要
本文记录Pulsar 2.2.0版本安装步骤，单机模式（standalone）比较简单，直接参考官网：http://pulsar.apache.org/docs/en/standalone/
按步骤执行就行，主要讲解集群模式安装，以及过程中遇到问题的解决。

### 0x2 环境要求
* Linux
* Java 8 及以上
* 3 台ZooKeeper集群

### 0x3 安装顺序
* 安装ZooKeeper集群
* 初始化集群元数据信息
* 安装BookKeeper集群
* 安装Pulsar brokers

下面针对每一步进行详细介绍。
### 0x4 安装ZooKeeper集群
由于我本地环境已经有安装好的ZK集群，可以直接使用，此步省略。

### 0x5 初始化集群元数据信息
初始化元数据信息非常简单，只需一条命令就可以，具体参数的意义看官网更好理解：
```
bin/pulsar initialize-cluster-metadata \
  --cluster pulsar-cluster-1 \
  --zookeeper zk1.us-west.example.com:2181 \
  --configuration-store zk1.us-west.example.com:2181 \
  --web-service-url http://pulsar.us-west.example.com:8080 \
  --web-service-url-tls https://pulsar.us-west.example.com:8443 \
  --broker-service-url pulsar://pulsar.us-west.example.com:6650 \
  --broker-service-url-tls pulsar+ssl://pulsar.us-west.example.com:6651
```
初始化成功会看到以下日志信息：
```
10:36:09.876 [main] INFO org.apache.bookkeeper.discover.ZKRegistrationManager - Successfully formatted BookKeeper metadata
10:36:09.880 [main] INFO org.apache.zookeeper.ZooKeeper - Session: 0x16734464b360002 closed
10:36:09.880 [main-EventThread] INFO org.apache.zookeeper.ClientCnxn - EventThread shut down for session: 0x16734464b360002
10:36:10.033 [main] INFO org.apache.pulsar.PulsarClusterMetadataSetup - Cluster metadata for 'pulsar-cluster-1' setup correctly
```
并且通过ZK客户端登录查看会看到以下节点信息：
```
[zk: localhost:2186(CONNECTED) 2] ls /
[zookeeper, counters, bookies, ledgers, managed-ledgers, schemas, namespace, admin, loadbalance]
```

此步非常重要，我在安装过程中忽略此步后启动BookKeeper直接报错，错误信息如下：
启动命令`bin/bookkeeper bookie`
```
10:26:17.185 [main] INFO org.apache.bookkeeper.proto.BookieNettyServer - Shutting down BookieNettyServer
10:26:17.196 [main] ERROR org.apache.bookkeeper.server.Main - Failed to build bookie server
org.apache.bookkeeper.bookie.BookieException$MetadataStoreException: Failed to get cluster instance id
 at org.apache.bookkeeper.discover.ZKRegistrationManager.getClusterInstanceId(ZKRegistrationManager.java:387) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.bookie.Bookie.checkEnvironmentWithStorageExpansion(Bookie.java:412) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.bookie.Bookie.checkEnvironment(Bookie.java:256) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.bookie.Bookie.<init>(Bookie.java:640) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.proto.BookieServer.newBookie(BookieServer.java:131) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.proto.BookieServer.<init>(BookieServer.java:100) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.server.service.BookieService.<init>(BookieService.java:43) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.server.Main.buildBookieServer(Main.java:299) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.server.Main.doMain(Main.java:219) [org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.server.Main.main(Main.java:201) [org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.proto.BookieServer.main(BookieServer.java:280) [org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
Caused by: org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = NoNode for BookKeeper metadata
 at org.apache.bookkeeper.discover.ZKRegistrationManager.getClusterInstanceId(ZKRegistrationManager.java:377) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 ... 10 more
```
验证BookKeeper时报以下错误：
验证命令`bin/bookkeeper shell bookiesanity`
```
09:49:14.430 [main] INFO org.apache.bookkeeper.client.BookKeeper - Weighted ledger placement is not enabled
09:49:14.489 [main] ERROR org.apache.bookkeeper.client.BookieWatcher - Failed to get bookie list : 
org.apache.bookkeeper.client.BKException$ZKException: Error while using ZooKeeper
 at org.apache.bookkeeper.discover.ZKRegistrationClient.lambda$getChildren$0(ZKRegistrationClient.java:212) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.bookkeeper.zookeeper.ZooKeeperClient$25$1.processResult(ZooKeeperClient.java:1174) ~[org.apache.bookkeeper-bookkeeper-server-4.7.2.jar:4.7.2]
 at org.apache.zookeeper.ClientCnxn$EventThread.processEvent_aroundBody0(ClientCnxn.java:604) ~[org.apache.pulsar-pulsar-broker-2.2.0.jar:2.2.0]
 at org.apache.zookeeper.ClientCnxn$EventThread$AjcClosure1.run(ClientCnxn.java:1) ~[org.apache.pulsar-pulsar-broker-2.2.0.jar:2.2.0]
 at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:149) ~[org.aspectj-aspectjrt-1.9.1.jar:?]
 at org.apache.pulsar.broker.zookeeper.aspectj.ClientCnxnAspect.timedProcessEvent(ClientCnxnAspect.java:72) ~[org.apache.pulsar-pulsar-broker-2.2.0.jar:2.2.0]
 at org.apache.zookeeper.ClientCnxn$EventThread.processEvent(ClientCnxn.java:528) ~[org.apache.pulsar-pulsar-broker-2.2.0.jar:2.2.0]
 at org.apache.zookeeper.ClientCnxn$EventThread.run(ClientCnxn.java:508) ~[org.apache.pulsar-pulsar-broker-2.2.0.jar:2.2.0]
Exception in thread "main" org.apache.bookkeeper.client.BKException$ZKException: Error while using ZooKeeper
 at org.apache.bookkeeper.discover.ZKRegistrationClient.lambda$getChildren$0(ZKRegistrationClient.java:212)
 at org.apache.bookkeeper.zookeeper.ZooKeeperClient$25$1.processResult(ZooKeeperClient.java:1174)
 at org.apache.zookeeper.ClientCnxn$EventThread.processEvent_aroundBody0(ClientCnxn.java:604)
 at org.apache.zookeeper.ClientCnxn$EventThread$AjcClosure1.run(ClientCnxn.java:1)
 at org.aspectj.runtime.reflect.JoinPointImpl.proceed(JoinPointImpl.java:149)
 at org.apache.pulsar.broker.zookeeper.aspectj.ClientCnxnAspect.timedProcessEvent(ClientCnxnAspect.java:72)
 at org.apache.zookeeper.ClientCnxn$EventThread.processEvent(ClientCnxn.java:528)
 at org.apache.zookeeper.ClientCnxn$EventThread.run(ClientCnxn.java:508)
```

### 0x6 安装BookKeeper集群
直接使用Pulsar内嵌的BK，单独安装未尝试。
修改$PULSAR_HOME/conf/bookkeeper.conf配置文件中的`zkServers`属性，如：
```
zkServers=zk1.us-west.example.com:2181,zk2.us-west.example.com:2181,zk3.us-west.example.com:2181
```
修改完成后直接命令启动，启动命令`bin/pulsar-daemon start bookie`
启动完成后使用命令`bin/bookkeeper shell bookiesanity`验证是否成功，看到以下信息说明启动成功
```
10:44:49.735 [main] INFO org.apache.zookeeper.ZooKeeper - Session: 0x46734464bc90004 closed
10:44:49.735 [main] INFO org.apache.bookkeeper.bookie.BookieShell - Bookie sanity test succeeded
10:44:49.735 [main-EventThread] INFO org.apache.zookeeper.ClientCnxn - EventThread shut down for session: 0x46734464bc90004
```
### 0x7 安装Pulsar brokers
修改$PULSAR_HOME/conf/broker.conf配置文件
```
zookeeperServers=zk1.us-west.example.com:2181,zk2.us-west.example.com:2181,zk3.us-west.example.com:2181
configurationStoreServers=zk1.us-west.example.com:2181,zk2.us-west.example.com:2181,zk3.us-west.example.com:2181
clusterName=pulsar-cluster-1
```
修改完成后使用命令`bin/pulsar-daemon start broker`启动broker
启动完成后可以通过`bin/pulsar-client produce`命令发送消息测试，我自己通过Java Client代码测试。
Producer代码：
```java
PulsarClient client = PulsarClient.builder().serviceUrl("pulsar://pulsar.us-west.example.com:6650").build();
Producer<String> producer = client.newProducer(Schema.STRING).topic("my-topic").create();
producer.send("aaaaaaaaaaaaaa");
producer.close();
```

Consumer代码：
```java
PulsarClient client = PulsarClient.builder().serviceUrl("pulsar://pulsar.us-west.example.com:6650").build();
Consumer<String> consumer = client.newConsumer(Schema.STRING)
 .topic("my-topic")
 .subscriptionName("my-topic-group-1")
 .subscribe();
do {
 Message<String> receive = consumer.receive();
 System.out.println(receive.getValue());
} while (true);
```

参考：http://pulsar.apache.org/docs/latest/deployment/cluster/