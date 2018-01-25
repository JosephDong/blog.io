---
layout: get
title: HDFS FileSystem 源码分析
date: 2018-01-25
categories: Hadoop
tags: [大数据,Hadoop,HDFS]
description: 理解FileSystem初始化过程。
---

###简介
FileSystem 是一个相当通用的文件系统的抽象类，负责文件系统相关操作，如：创建目录、创建文件、删除目录或文件、读取文件内容等。既然是抽象类，肯定有具体的实现，如下图：
![](index_files/2df98983-d741-4ff2-9c25-859633ecc19a.png)
本文主要讲解 HDFS 分布式文件系统，具体实现类为`DistributedFileSystem`。
 
###创建 FileSystem 实例源码分析
下面以`FileSystem fs = FileSystem.get(new Configuration());`为例进行源码剖析。
*进入`get`方法，源码如下：*
```java
/**
  * Returns the configured filesystem implementation.
  * @param conf the configuration to use
  */
public static FileSystem get(Configuration conf) throws IOException {
   return get(getDefaultUri(conf), conf);
}
```
`getDefaultUri`方法源码如下：
```java
/** Get the default filesystem URI from a configuration.
 * @param conf the configuration to use
 * @return the uri of the default filesystem
 */
public static URI getDefaultUri(Configuration conf) {
  return URI.create(fixName(conf.get(FS_DEFAULT_NAME_KEY, DEFAULT_FS)));
}
```
此方法通过配置文件中参数`fs.defaultFS`指定的值来生成URI对象，如果配置文件中没有指定，则读取默认值为`file:///`，即本地文件系统。
继续进入`get(uri, conf)`方法分析源码：
```java
/** Returns the FileSystem for this URI's scheme and authority.  The scheme
 * of the URI determines a configuration property name,
 * <tt>fs.<i>scheme</i>.class</tt> whose value names the FileSystem class.
 * The entire URI is passed to the FileSystem instance's initialize method.
 */
public static FileSystem get(URI uri, Configuration conf) throws IOException {
  String scheme = uri.getScheme();
  String authority = uri.getAuthority();

  if (scheme == null && authority == null) {     // use default FS
    return get(conf);
  }

  if (scheme != null && authority == null) {     // no authority
    URI defaultUri = getDefaultUri(conf);
    if (scheme.equals(defaultUri.getScheme())    // if scheme matches default
        && defaultUri.getAuthority() != null) {  // & default has authority
      return get(defaultUri, conf);              // return default
    }
  }
  
  String disableCacheName = String.format("fs.%s.impl.disable.cache", scheme);
  if (conf.getBoolean(disableCacheName, false)) {
    return createFileSystem(uri, conf);
  }

  return CACHE.get(uri, conf);
}
```
第`22~25`行判断是否屏蔽缓存功能，默认情况是开户缓存功能，如果屏蔽缓存功能，则每次都会新创建一个连接，不推荐这样做。
第`27`从CACHE中获取FileSystem实例，下面进入`CACHE.get()`方法：
```java
FileSystem get(URI uri, Configuration conf) throws IOException{
  Key key = new Key(uri, conf);
  return getInternal(uri, conf, key);
}
```
进入`getInternal`方法：
```java
private FileSystem getInternal(URI uri, Configuration conf, Key key) throws IOException{
  FileSystem fs;
  synchronized (this) {
    fs = map.get(key);
  }
  if (fs != null) {
    return fs;
  }

  fs = createFileSystem(uri, conf);
  synchronized (this) { // refetch the lock again
    FileSystem oldfs = map.get(key);
    if (oldfs != null) { // a file system is created while lock is releasing
      fs.close(); // close the new file system
      return oldfs;  // return the old file system
    }
    
    // now insert the new file system into the map
    if (map.isEmpty()
            && !ShutdownHookManager.get().isShutdownInProgress()) {
      ShutdownHookManager.get().addShutdownHook(clientFinalizer, SHUTDOWN_HOOK_PRIORITY);
    }
    fs.key = key;
    map.put(key, fs);
    if (conf.getBoolean("fs.automatic.close", true)) {
      toAutoClose.add(key);
    }
    return fs;
  }
}
```
此方法的其他逻辑不做分析，核心就是从缓存中获取`fs`，如果获取为空则通过`createFileSystem`方法新创建实例，创建成功后将实例放入缓存中。
我们重点看一下`createFileSystem`方法看源码实现：
```java
private static FileSystem createFileSystem(URI uri, Configuration conf
    ) throws IOException {
  Class<?> clazz = getFileSystemClass(uri.getScheme(), conf);
  if (clazz == null) {
    throw new IOException("No FileSystem for scheme: " + uri.getScheme());
  }
  FileSystem fs = (FileSystem)ReflectionUtils.newInstance(clazz, conf);
  fs.initialize(uri, conf);
  return fs;
}
```
通过`getFileSystemClass`方法获取文件系统Class对象，获取Class对象后通过反射机制创建FileSystem实例，下面看一下`getFileSystemClass`方法源码实现：
```java
public static Class<? extends FileSystem> getFileSystemClass(String scheme,
    Configuration conf) throws IOException {
  if (!FILE_SYSTEMS_LOADED) {
    loadFileSystems();
  }
  Class<? extends FileSystem> clazz = null;
  if (conf != null) {
    clazz = (Class<? extends FileSystem>) conf.getClass("fs." + scheme + ".impl", null);
  }
  if (clazz == null) {
    clazz = SERVICE_FILE_SYSTEMS.get(scheme);
  }
  if (clazz == null) {
    throw new IOException("No FileSystem for scheme: " + scheme);
  }
  return clazz;
}
```
1、首先判断是否已经初始化加载过，如果没有，则调用`loadFileSystems`方法初始所有文件系统，并缓存。
2、优先从配置文件中指定的文件系统，通过`fs.[scheme].impl`参数指定。
3、如果配置文件中没有指定，则通过 scheme 直接从缓存中获取。

回到`createFileSystem`方法源码，通过反射实例化`fs`对象后，调用`initialize`方法做初始化工作，下面看一下`DistributedFileSystem.initialize`方法的实现：
```java
@Override
public void initialize(URI uri, Configuration conf) throws IOException {
  super.initialize(uri, conf);
  setConf(conf);

  String host = uri.getHost();
  if (host == null) {
    throw new IOException("Incomplete HDFS URI, no host: "+ uri);
  }
  homeDirPrefix = conf.get(
      DFSConfigKeys.DFS_USER_HOME_DIR_PREFIX_KEY,
      DFSConfigKeys.DFS_USER_HOME_DIR_PREFIX_DEFAULT);
  
  this.dfs = new DFSClient(uri, conf, statistics);
  this.uri = URI.create(uri.getScheme()+"://"+uri.getAuthority());
  this.workingDir = getHomeDirectory();
}
```
此方法最核心的代码是第`14`行初始化`DFSClient`类，所有文件系统相关操作全部在此类中实现，下篇文章再分享里面的实现细节。紧接着返回`fs`实例，到此整个FileSystem实例初始化完成。
