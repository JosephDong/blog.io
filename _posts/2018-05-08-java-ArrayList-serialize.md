---
layout: post
title: ArrayList 序列化分析
date: 2018-05-07
categories: Java
tags: [ArrayList,序列化]
description: ArrayList序列化、反序列化原理解析。
---

### 0x1 摘要
相信ArrayList是Java开发过程中最常用的集合类之一，底层存储结构是数组，这篇文章不讲解底层数据结构的实现，主要讲解它的序列化机制，大家都知道ArrayList是可以序列化的，但也有一些人不知道具体是怎么序列化的，希望这篇章可以帮到大家。

### 0x2 ArrayList 类结构图
![ArrayList UML图](https://raw.githubusercontent.com/yuesefu/yuesefu.github.io/master/img/custorm/ArrayList_UML.png)
从类结构图可以看出实现了Serializable接口，说明是可以被序列化的。

### 0x3 序列化过程详解
在讲解序列过程前，先要清楚ArrayList底层数据结构，摘要中已经提到底层是通过数组实现，下面从源码看一下：
```java
/**
* The array buffer into which the elements of the ArrayList are stored.
* The capacity of the ArrayList is the length of this array buffer. Any
* empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
* will be expanded to DEFAULT_CAPACITY when the first element is added.
*/
transient Object[] elementData; // non-private to simplify nested class access
```
`elementData`成员变量就是用来存储元素，细心的同学可能已经发现，此成员变量用`transient`关键字修饰，而`transient`关键字的作用是不需要序列化，存储数据的数组不序列化，那ArrayList是怎么实现数据序列化的呢？
继续翻源码会发现存在`writeObject`、`readObject`两个方法，
```java
/**
* Save the state of the <tt>ArrayList</tt> instance to a stream (that
* is, serialize it).
*
* @serialData The length of the array backing the <tt>ArrayList</tt>
*            instance is emitted (int), followed by all of its elements
*            (each an <tt>Object</tt>) in the proper order.
*/
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

/**
* Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
* deserialize it).
*/
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;

    // Read in size, and any hidden stuff
    s.defaultReadObject();

    // Read in capacity
    s.readInt(); // ignored

    if (size > 0) {
        // be like clone(), allocate array based upon size not capacity
        ensureCapacityInternal(size);

        Object[] a = elementData;
        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}
```
从源码上可以看出，从数组中按`size`大小将元素取出来挨个写入到`ObjectOutputStream`中，反序列化时首先从`ObjectInputStream`中取得`size`大小，再循环取得元素。
正常情况下，针对要序列化的对象不需要定义`writeObject`、`readObject`方法，采用默认序列化、反序列化方式，如果有特殊需求，则可以在对象中定义`writeObject`、`readObject`方法进行扩展，详细实现原理下次写序列化文章时再介绍。
那么，ArrayList为什么要采用这种机制呢？
上文中已经提到ArrayList底层采用数组存储，当声明一个数组时长度是固定的，在ArrayList中遇到元素个数超过数组大小时会采用自动扩展的方式来加大数组容量，而扩展方法是在原有数组容量的基础上扩大到1.5倍，具体实现可以查看`ArrayList#grow`方法。这种扩展方法会带来底层数组会存在空值问题，也就是真实数组容量远大于实际元素个数，在这样的前提下，如果直接序列`elementData`数组会带来不必要的开销，序列化很多空值，从而降低了序列化、反序列化性能，所以通过`writeObject`、`readObject`方法来实现序列化实际的元素，从而提升性能。
