---
layout: post
title: Java 动态代理
date: 2018-01-24
categories: Java
tags: [静态代理,动态代理]
description: 一文读懂Java动态代理技术。
---

### 0x1 简介
相信大家对代理都不陌生，就算在项目中没有实际用到过，那你肯定也听说过代理模式，在开源组件中使用也非常广泛，如：Spring AOP功能。那今天为什么还特地来讲Java动态代理模式呢，因为近期在看Hadoop RPC源代码时发现，Hadoop RPC 就采用动态代理模式来实现，为了更好的理解Hadoop RPC底层实现，先来温故一下动态代理技术。

如果你对Java动态代理技术非常了解，并知道其底层实现原理，那你可以不用看这篇文章。
如果你对Java动态代理技术有一定了解，并知道如何使用，但不知道底层实现原理，那这篇文章值得你花时间看一下。
如果你对Java动态代理技术不了解，那你也可以通过对这篇文章的学习后，让你可以在项目中使用Java动态代理技术。

先思考一下 *什么是代理？* 
参考百度汉语解释：http://hanyu.baidu.com/zici/s?wd=%E4%BB%A3%E7%90%86&query=%E4%BB%A3%E7%90%86&srcid=28232&from=kg0&from=kg0
代理是指：受委托代表当事人进行某种活动。如：你委托律师打官司、市长委托秘书做某些事情等等。

Java中分静态代理和动态代理，本文主要讲解动态代理的实现原理。相对动态代理来讲，静态代理的实现更加简单易懂，为了更好的理解动态代理，我们首先来看一下静态代理的实现例子。

### 0x2 静态代理介绍
*接口：*
```java
/**
 * 登录服务接口
 */
public interface LoginService {
    void login(String username, String password);
}
```

*目标实现类：*
```java
/**
 * 登录服务实现类
 */
public class LoginServiceImpl implements LoginService {
    @Override
    public void login(String username, String password) {
        System.out.printf("当前登录用户为：%s \n" ,username);
        //TODO 登录处理
    }
}
```
*代理类：*
```java
/**
 * 登录服务代理类
 */
public class LoginServiceProxy implements LoginService{
    private LoginService loginService;

    public LoginServiceProxy(LoginService target){
        this.loginService = target;
    }

    @Override
    public void login(String username, String password) {
        System.out.println("登录开始，并记录当前时间...");
        this.loginService.login(username, password);
        System.out.println("登录结束，并记录当前时间...");
    }
}
```
*测试类：*
```java
public class Main {
    public static void main(String[] args) {
        LoginServiceImpl target = new LoginServiceImpl();
        LoginServiceProxy proxy = new LoginServiceProxy(target);
        proxy.login("Joseph", "abc123");
    }
}
```
*运行结果：*
```
登录开始，并记录当前时间...
当前登录用户为：Joseph 
登录结束，并记录当前时间...
```
从运行结果可以看出，我们给目标方法调用增加了开始和结束日志打印，而这样的场景也是代理模式中比较常见的。

**优缺点总结：**
优点：不用修改目标对象功能的前提下，实现功能扩展，如：上例子中针对登录动作前后做了日志记录。
缺点：
* 代理类必须实现接口，导致代理类太多；
* 一旦接口发生变更，代理类也必须同步更新，导致维护成本会比较高；

### 0x3 动态代理介绍
动态代理中的`动态`体现在哪里呢？
静态代码是事先通过Java代码已经实现好，而动态是在运行期通过JDK API在内存中构建代理对象。通过动态代理可以很方便的实现为目标对象添加功能，如：为目标对象的所有方法都增加方法耗时记录、方法调用日志记录等功能。

**动态代理实现原理介绍：**
Java 的动态机制有两个重要的接口或类：
* InvocationHandler（接口）
* Proxy（类）

下面来看一下JDK API中的介绍：
**InvocationHandler接口：**
```
/**
 * {@code InvocationHandler} is the interface implemented by
 * the <i>invocation handler</i> of a proxy instance.
 *
 * <p>Each proxy instance has an associated invocation handler.
 * When a method is invoked on a proxy instance, the method
 * invocation is encoded and dispatched to the {@code invoke}
 * method of its invocation handler.
 *
 * @author      Peter Jones
 * @see         Proxy
 * @since       1.3
 */
public interface InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```
一个代理实例调用处理的接口实现，所有代理实例必须实现`InvocationHandler`接口，当在代理实例上调用方法时，方法调用将会被派发给`InvocationHandler`实现的`invoke`方法处理。下面分析一下`invoke`方法的参数及返回值：
*参数：*
`proxy`：指我们的代理对象，即通过JDK API自动生成的对象
`method`：我们所要调用真实对象的某个方法的Method对象
`args`：调用真实对象某个方法时接受的参数

*返回值：*
Object：调用代理对象方法的返回值

注：光通过参数的解释可能并不是很明白，没关系，下面会通过实例对参数进行详细介绍。

**Proxy类：**
Proxy类提供了很多方法，但我们平时经常用到的是`newProxyInstance `方法，所以重点介绍此方法，先看一方法参数的介绍：
```
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h) 
throws IllegalArgumentException
```
*参数：*
`loader`：由指定的ClassLoader对象来加载生成的代理对象
`interfaces`：一个接口数组，指代理对象需要实现的接口列表，如果指定了接口，那么代理对象将实现指定的接口，这样我们就可以通过接口来调用方法
`h`：一个InvocationHander对象，表示调用代理对象方法的时候会调用哪一个InvocationHander对象的invoke方法

*返回值：*
Object：动态生成的代理对象

下面通过一个实例来讲解上面提到的`InvocationHandler `接口和`Proxy`类的功能，例子实现的功能跟上面静态代理的功能一致，方便进行对比：
**动态代理实例代码**
*接口：*
```
public interface LoginService {
    void login(String username, String password);
}
```
*实现类：*
```
public class LoginServiceImpl implements LoginService {
    @Override
    public void login(String username, String password) {
        System.out.printf("当前登录用户为：%s \n" ,username);
        //TODO 登录处理
    }
}
```
*代理类：*
```
public class LoginServiceProxy implements InvocationHandler {
    private Object obj;

    public LoginServiceProxy(Object obj){
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("登录开始，并记录当前时间...");
        Object result = method.invoke(obj, args);
        System.out.println("登录结束，并记录当前时间...");
        return result;
    }
}
```
*测试类：*
```
public class Main {
    public static void main(String[] args) {
        LoginService loginService = (LoginService) Proxy.newProxyInstance(LoginService.class.getClassLoader(), new
                Class[]{LoginService.class}, new LoginServiceProxy(new LoginServiceImpl()));
        loginService.login("Joseph", "test");
    }
}
```
*运行结果：*
```
登录开始，并记录当前时间...
当前登录用户为：Joseph 
登录结束，并记录当前时间...
```
从上面代码及运行结果来，有没有发现动态代理和静态代理有哪些不同点呢？粗略看着没什么区别，目前来看的确是没多大区别，也没看出动态代理的优势在哪，下面我们详细分析一下。
首先，我们来假设一个场景，比如：不改变原有方法源码的情况下，你想记录项目中部分类的方法调用耗时（前提是你需要记录的类是有接口实现的）。

假如需要记录方法耗时的类有10个，那么用静态代码方法你需要实现10个代理类，因为静态代理必须去实现原有类的接口来实现，如果用动态代理就非常简单，只需要实现一个代理类（实现InvocationHandler 接口）就可以完成10个类的方法耗时记录，大降低了开发的代码量。

到这里，也许有人会问：动态代理是实现如何的呢？
动态代码的关键就在于`动态`两个字，意思就是JVM会在运行时动态去创建代理类，看上面`测试类：`代码块，其中`Proxy.newProxyInstance`就实现了动态创建代理的功能，为了更好的理解这个功能，我们把测试类稍微做改动，如下：
```
public class Main {
    public static void main(String[] args) {
        LoginService loginService = (LoginService) Proxy.newProxyInstance(LoginService.class.getClassLoader(), new
                Class[]{LoginService.class}, new LoginServiceProxy(new LoginServiceImpl()));
        System.out.println(loginService.getClass().getName());
        loginService.login("Joseph", "test");
    }
}
```
加了第`5`代码，我们再来看看运行结果：
```
com.sun.proxy.$Proxy0
登录开始，并记录当前时间...
当前登录用户为：Joseph 
登录结束，并记录当前时间...
```
有人会问怎么打印出来是`$Proxy0`，没错，这就是JVM自动生成的代理类。那么，新的问题来了：
1、`$Proxy0`代理类和`LoginService`接口有什么关系？
2、`$Proxy0`代理类和`LoginServiceImpl `实现类有什么关系？
3、`$Proxy0`代理类和`LoginServiceProxy `类有什么关系？

我们带着这三个问题继续思考，怎么才能很好的解答这三个问题呢，源码、源码、源码，重要的事情说三遍。下面我们看看创建代码的源码，从`Proxy.newProxyInstance`入口开始，
```
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
{
	Objects.requireNonNull(h);

	final Class<?>[] intfs = interfaces.clone();
	final SecurityManager sm = System.getSecurityManager();
	if (sm != null) {
		checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
	}

	/*
	 * Look up or generate the designated proxy class.
	 */
	Class<?> cl = getProxyClass0(loader, intfs);

	/*
	 * Invoke its constructor with the designated invocation handler.
	 */
	try {
		if (sm != null) {
			checkNewProxyPermission(Reflection.getCallerClass(), cl);
		}

		final Constructor<?> cons = cl.getConstructor(constructorParams);
		final InvocationHandler ih = h;
		if (!Modifier.isPublic(cl.getModifiers())) {
			AccessController.doPrivileged(new PrivilegedAction<Void>() {
				public Void run() {
					cons.setAccessible(true);
					return null;
				}
			});
		}
		return cons.newInstance(new Object[]{h});
	} catch (IllegalAccessException|InstantiationException e) {
		throw new InternalError(e.toString(), e);
	} catch (InvocationTargetException e) {
		Throwable t = e.getCause();
		if (t instanceof RuntimeException) {
			throw (RuntimeException) t;
		} else {
			throw new InternalError(t.toString(), t);
		}
	} catch (NoSuchMethodException e) {
		throw new InternalError(e.toString(), e);
	}
}
```
重点关注第`17`行代码，其余代码不做分析，继续进入`getProxyClass0`方法看源码：
```
private static Class<?> getProxyClass0(ClassLoader loader,
									   Class<?>... interfaces) {
	if (interfaces.length > 65535) {
		throw new IllegalArgumentException("interface limit exceeded");
	}

	// If the proxy class defined by the given loader implementing
	// the given interfaces exists, this will simply return the cached copy;
	// otherwise, it will create the proxy class via the ProxyClassFactory
	return proxyClassCache.get(loader, interfaces);
}
```
这个方法非常简单，先做接口列表长度判断，超出`65535`个接口直接抛出异常，不过哪类会实现这么多接口？我们不关心这个，直接看第`10`返回代码，从缓存中获取对象，看一下`proxyClassCache `缓存的定义，
```
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```
重点关注`ProxyClassFactory `代理类工厂类中的`apply`方法，
```
@Override
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
	//代码省略
	
	/*
	 * Generate the specified proxy class.
	 */
	byte[] proxyClassFile = ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags);
	try {
		return defineClass0(loader, proxyName, proxyClassFile, 0, proxyClassFile.length);
	} catch (ClassFormatError e) {
		/*
		 * A ClassFormatError here means that (barring bugs in the
		 * proxy class generation code) there was some other
		 * invalid aspect of the arguments supplied to the proxy
		 * class creation (such as virtual machine limitations
		 * exceeded).
		 */
		throw new IllegalArgumentException(e.toString());
	}
}
```
由于源代码较多，直接省略无关代码，直接看第`8`行核心代码，通过`ProxyGenerator.generateProxyClass`方法生成代理类的字节数组，再调用第`10`行`defineClass0`本地方法返回Class对象。
上术代码中最关键的是`ProxyGenerator.generateProxyClass`方法，今天不讲这个方法的实现，我们利用这个方法来生成二进制文件来分析代码类的源码，我们再改一下`测试类`代码，如下：
```
public class Main {
    public static void main(String[] args) throws IOException {
        LoginService loginService = (LoginService) Proxy.newProxyInstance(LoginService.class.getClassLoader(), new
                Class[]{LoginService.class}, new LoginServiceProxy(new LoginServiceImpl()));
        System.out.println(loginService.getClass().getName());
        loginService.login("Joseph", "test");

        //生成代理类二进制文件
        String className = "ProxyTest";
        byte[] data = ProxyGenerator.generateProxyClass(className, new Class[]{LoginService.class});
        FileOutputStream out = new FileOutputStream(className + ".class");
        out.write(data);
        out.close();
    }
}
```
运行后，会在工程目录下生成`ProxyTest.class`文件，

直接用idea打开，源码如下：
```
public final class ProxyTest extends Proxy implements LoginService {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public ProxyTest(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void login(String var1, String var2) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1, var2});
        } catch (RuntimeException | Error var4) {
            throw var4;
        } catch (Throwable var5) {
            throw new UndeclaredThrowableException(var5);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("com.joseph.java.proxy.dynamicproxy.LoginService").getMethod("login", new Class[]{Class.forName("java.lang.String"), Class.forName("java.lang.String")});
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
通过生成的源代码就不难回答上面提出的3个问题了，
1、代理类实现了`LoginSerivce`接口，并实现`login`方法
2、代理类和真正的接口实现类`LoginServiceImpl `没有直接关系
3、`LoginServiceProxy`类作为构造函数参数传入代理类中，并在业务方法`login`中调用`LoginServiceProxy`类的`invoke`方法

到这里已经把Java动态代理的原理讲完了，从怎么使用到动态代理创建的过程，但生成动态代理类字节数组的源码没有详细分析，这个留给大家自己去分析源码，`ProxyGenerator.generateProxyClass`这个方法中，主要思想就是反映机制来生成类及方法。

### 0x4 总结
Java动态代理相比静态代理比较方便灵活、具有扩展性，使用起来也没有难度，所以建议在项目中使用动态代理。
那动态代理有没有不足的地方呢？
动态代理是针对接口实现的，不能对类进行代理，我也不知道这算不算缺点，至少是一种限制吧。

