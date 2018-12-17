---
layout: post
title: Flink 事件时间的陷进及解决思路
date: 2018-12-17
categories: Flink
tags: [Flink,水印,事件时间]
description: Flink 事件时间的陷进及解决思路

---

### 0x1 摘要
大家都知道Flink引入了事件时间（eventTime）这个重要概念，来提升数据统计的准确性，但引入事件时间后在具体业务实现时存在一些问题必需要合理去解决，否则会造成非常严重的问题。

### 0x2 Flink 时间概念介绍
Flink 支持不同的时间概念，包括：
* Event Time ：事件时间
* Processing Time ：处理时间
* Ingestion Time ：消息提取时间

参考下图可以清晰的知道这三者的关系：
![时间概念图](https://ci.apache.org/projects/flink/flink-docs-release-1.5/fig/times_clocks.svg)
`Ingestion Time`是介于`Event Time`和`Processing Time`之间的概念。
程序中可以通过`env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);`指定使用时间类型。

### 0x3 事件时间存在的问题
事件时间存在什么样的问题呢？下面先看一个简单的业务场景。
比如：要统计APP上搜索按钮每1分钟的点击次数。
前端埋点数据结构：

|字段名|字段类型|描述|
|:---|:---|:---|
|eventCode|String|事件编码|
|clickTime|Long|点击时间|

基于以上数据结构我们可设计如下水印处理器：
```java
public static class TimestampExtractor implements AssignerWithPeriodicWatermarks<Tuple2<String, Long>> {

 private long currentMaxTimestamp = 0L;

 @Override
 public Watermark getCurrentWatermark() {
  return new Watermark(currentMaxTimestamp -3000);
 }

 @Override
 public long extractTimestamp(Tuple2<String, Long> tuple, long previousElementTimestamp) {
  long eventTime = tuple.f1;
  currentMaxTimestamp = Math.max(currentMaxTimestamp, eventTime);
  return eventTime;
 }
}
```
`extractTimestamp`方法会拿事件时间和上一次事件时间比较，并取较大值来更新当前水印值。
假设前端发送了以下这些数据，方便直观看数据clickTime直接采用格式化后的值，并以逗号分隔数据。
```
001,2018-12-17 13:30:00
001,2018-12-17 13:30:01
001,2018-12-17 13:30:02
001,2018-12-18 13:30:00
001,2018-12-17 13:30:03
001,2018-12-17 13:30:04
001,2018-12-17 13:30:05
```
正常数据都是17号，突然来了一条18号的数据，再结合上面的水印逻辑，一旦出现这种问题数据，直接导致水位上升到18号，后面再来17号的数据全部无法处理。针对业务来讲这样的错误是致命的，统计结果出现断层。

### 0x4 解决思路
针对以上问题我们可以对水印实现类做如下改造：
```java
public static class TimestampExtractor implements AssignerWithPeriodicWatermarks<Tuple2<String, Long>> {

 private long currentMaxTimestamp = 0L;

 @Override
 public Watermark getCurrentWatermark() {
  return new Watermark(currentMaxTimestamp -3000);
 }

 @Override
 public long extractTimestamp(Tuple2<String, Long> tuple, long previousElementTimestamp) {
  long eventTime = tuple.f1;
  if((currentMaxTimestamp == 0) || (eventTime - currentMaxTimestamp < MESSAGE_FORWARD_TIME)) {
            currentMaxTimestamp = Math.max(currentMaxTimestamp, eventTime);
        }
  return eventTime;
 }
}
```
`MESSAGE_FORWARD_TIME`变量是自定义的消息最大跳跃时间，如果超出这个范围则不更新最大水印时间。