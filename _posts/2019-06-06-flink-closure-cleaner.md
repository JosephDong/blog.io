---
layout: post
title: Flink 闭包清除源码分析
date: 2019-06-05
categories: Flink
tags: [Flink,序列化,闭包清除]
description: Flink 闭包清除源码分析

---

### 0x1 摘要
本文主要讲解Flink里为什么需要做闭包清除？Flink是怎么实现闭包清除的？

### 0x2 Flink 为什么要做闭包清除
大家都知道Flink中算子都是通过序列化分发到各节点上，所以要确保算子对象是可以被序列化的，很多时候大家比较喜欢直接用匿名内部类实现算子，而匿名内部类就会带来闭包问题，当匿名内部类引用的外部对象没有实现序列化接口时，就会导致内部类无法被序列化，因此Flink框架底层必须做好清除工作。


### 0x3 Flink 闭包清除实现
先来看一个Map算子代码：
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
final DataStreamSource<String> source = env.addSource(new SourceFunction<String>() {
    @Override
    public void run(SourceContext<String> ctx) throws Exception {

    }

    @Override
    public void cancel() {

    }
});
source.map(new MapFunction<String, String>() {
    @Override
    public String map(String value) throws Exception {
        return null;
    }
});
```
跟进源码查看map方法:
```
public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {
	TypeInformation<R> outType = TypeExtractor.getMapReturnTypes(clean(mapper), getType(),
			Utils.getCallLocationName(), true);
	return transform("Map", outType, new StreamMap<>(clean(mapper)));
}
```
重点关注`clean(mapper)`代码，继续跟进源码，最终会走到`StreamExecutionEnvironment`类的以下方法：
```
@Internal
public <F> F clean(F f) {
	if (getConfig().isClosureCleanerEnabled()) {
		ClosureCleaner.clean(f, true);
	}
	ClosureCleaner.ensureSerializable(f);
	return f;
}
```
到这里已经可以看出来闭包清除工具类`ClosureCleaner`，下面我们详细剖析一下此类。
先看`clean`方法：
```
public static void clean(Object func, boolean checkSerializable) {
	if (func == null) {
		return;
	}

	final Class<?> cls = func.getClass();

	// First find the field name of the "this$0" field, this can
	// be "this$x" depending on the nesting
	boolean closureAccessed = false;

	for (Field f: cls.getDeclaredFields()) {
		if (f.getName().startsWith("this$")) {
			// found a closure referencing field - now try to clean
			closureAccessed |= cleanThis0(func, cls, f.getName());
		}
	}

	if (checkSerializable) {
		try {
			InstantiationUtil.serializeObject(func);
		}
		catch (Exception e) {
			String functionType = getSuperClassOrInterfaceName(func.getClass());

			String msg = functionType == null ?
					(func + " is not serializable.") :
					("The implementation of the " + functionType + " is not serializable.");

			if (closureAccessed) {
				msg += " The implementation accesses fields of its enclosing class, which is " +
						"a common reason for non-serializability. " +
						"A common solution is to make the function a proper (non-inner) class, or " +
						"a static inner class.";
			} else {
				msg += " The object probably contains or references non serializable fields.";
			}

			throw new InvalidProgramException(msg, e);
		}
	}
}
```
方法参数：
* `func`：要清除的对应
* `checkSerializable`：清除完成后是否需要调用序列方法进行验证

第一步：查找闭包引用的成员变量，通过反射判断成员变量名是否包含`this$`来判定，代码片断：
```
for (Field f: cls.getDeclaredFields()) {
	if (f.getName().startsWith("this$")) {
		// found a closure referencing field - now try to clean
		closureAccessed |= cleanThis0(func, cls, f.getName());
	}
}
```
找到闭包引用的成员变量后，调用内部私有方法`cleanThis0`方法处理，看方法源码：
```
private static boolean cleanThis0(Object func, Class<?> cls, String this0Name) {
	This0AccessFinder this0Finder = new This0AccessFinder(this0Name);
	getClassReader(cls).accept(this0Finder, 0);

	final boolean accessesClosure = this0Finder.isThis0Accessed();

	if (LOG.isDebugEnabled()) {
		LOG.debug(this0Name + " is accessed: " + accessesClosure);
	}

	if (!accessesClosure) {
		Field this0;
		try {
			this0 = func.getClass().getDeclaredField(this0Name);
		} catch (NoSuchFieldException e) {
			// has no this$0, just return
			throw new RuntimeException("Could not set " + this0Name + ": " + e);
		}

		try {
			this0.setAccessible(true);
			this0.set(func, null);
		}
		catch (Exception e) {
			// should not happen, since we use setAccessible
			throw new RuntimeException("Could not set " + this0Name + " to null. " + e.getMessage(), e);
		}
	}

	return accessesClosure;
}
```
核心代码`this0.set(func, null);`将闭包引用置空处理，此方法还用到了ASM包，具体逻辑没完成整明白。