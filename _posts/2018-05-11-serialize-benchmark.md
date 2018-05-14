---
layout: post
title: 序列化、反序列化性能测试
date: 2018-05-11
categories: Java
tags: [序列化,Java,Avro,Thrift,Protobuf]
description: 序列化、反序列化性能测试
---

### 0x1 摘要
平时开发中经常听到序列与反序列化，特别是在分布式系统与RPC应用中，今天突然心血来潮对几种常用的序列化框架做个性能测试对比，测试对象：
* Java 原生序列
* Avro
* Thrift
* Protobuf

### 0x2 测试环境及工具
**测试环境：**
系统类型：64 位操作系统
CPU：Intel(R) Core(TM) i3-4130 CPU @ 3.40 GHz 3.40 GHz
内存：8 GB
开发工具：IDEA

**测试工具：**
JMH

### 0x3 测试实体
```java
public class User implements Serializable{
    private static final long serialVersionUID = 5149128310592716591L;

    private int age;
    private String username;
    private String address;

    public int getAge() {
        return age;
    }

    public User setAge(int age) {
        this.age = age;
        return this;
    }

    public String getUsername() {
        return username;
    }

    public User setUsername(String username) {
        this.username = username;
        return this;
    }

    public String getAddress() {
        return address;
    }

    public User setAddress(String address) {
        this.address = address;
        return this;
    }
}
```

### 0x4 JMH参数设置
```java
@BenchmarkMode(Mode.Throughput)
@Warmup(iterations = 3)
@Measurement(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
@Threads(4)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
```
参数意义不在此文做介绍，大家可以自行网上搜索或者看官网。

### 0x5 测试结果
**Encode：**

| Benchmark | Mode | Cnt | Score | Erorr | Units |
|--|--|--|--:|--:|--|
|SerializableEncodeTest.testAvroSerializableEncode|thrpt|20|2881.398 ±|27.619|ops/ms|
|SerializableEncodeTest.testJavaSerializableEncode|thrpt|20|1708.157 ±|122.516|ops/ms|
|SerializableEncodeTest.testProtobufSerializableEncode|thrpt|20|10349.962 ±|215.994|ops/ms|
|SerializableEncodeTest.testThriftSerializableEncode|thrpt|20|5292.191 ±|49.890|ops/ms|

**Decode：**

| Benchmark | Mode | Cnt | Score | Erorr | Units |
|--|--|--|--:|--:|--|
|SerializableEncodeTest.testAvroSerializableEncode|thrpt|20|931.167 ±|13.185|ops/ms|
|SerializableEncodeTest.testJavaSerializableEncode|thrpt|20|521.145 ±|6.993|ops/ms|
|SerializableEncodeTest.testProtobufSerializableEncode|thrpt|20|25335.295 ±|564.496|ops/ms|
|SerializableEncodeTest.testThriftSerializableEncode|thrpt|20|5239.726 ±|75.048|ops/ms|

### 0x6 结论
从结果中不难看出，`Protobuf`不管是序列化还是反序列化都有绝对优势，`Java原生`都是最弱的。
