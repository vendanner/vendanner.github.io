---
layout:     post
title:      Flink child-first类加载
subtitle:   
date:       2020-12-28
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
---

Flink 1.11

Java 的类加载机制是父类委派模型：如果一个类加载器要加载一个类，它首先不会自己尝试加载这个类，而是把加载的请求委托给父加载器完成，所有的类加载请求最终都应该传递给最顶层的启动类加载器。只有当父加载器无法加载到这个类时，子加载器才会尝试自己加载。

在 Flink 的上下文中，Flink 集群运行着 Flink 的框架代码，这些代码包括 Flink 的各种依赖。当用户代码使用到的 jar 与 Flink 已有的依赖**版本不一致**时，就很有可能发生 `NoSuchMethodError`、`IllegalAccessError` 等。因为根据 Java 的类加载机制是先加载 Flink 的依赖中的 jar 包。为了解决这个问题，Flink 提供 `classloader.resolve-order=child-first`策略允许先加载用户代码。

```java
// org.apache.flink.util.ChildFirstClassLoader#loadClassWithoutExceptionHandling
protected synchronized Class<?> loadClassWithoutExceptionHandling(
    String name,
    boolean resolve) throws ClassNotFoundException {

  // 已加载的不需要再次加载
  Class<?> c = findLoadedClass(name);

  if (c == null) {
    // 有些特殊的类，不能让用户的jar 加载(java、flink类，具体看下文)
    for (String alwaysParentFirstPattern : alwaysParentFirstPatterns) {
      if (name.startsWith(alwaysParentFirstPattern)) {
        return super.loadClassWithoutExceptionHandling(name, resolve);
      }
    }
    // 下面这段代码就是实现先加载用户代码的关键：
    //    先在自己 classload 加载
    //    没找到再从父类加载
    // 显然这是颠覆 Java 原有的类加载机制
    try {
      c = findClass(name);
    } catch (ClassNotFoundException e) {
      c = super.loadClassWithoutExceptionHandling(name, resolve);
    }
  }

  if (resolve) {
    resolveClass(c);
  }

  return c;
}
```

不允许用户代码加载的 Class

```java
java.;
scala.;
org.apache.flink.;
com.esotericsoftware.kryo;
org.apache.hadoop.;
javax.annotation.;
org.slf4j;
org.apache.log4j;
org.apache.logging;
org.apache.commons.logging;
ch.qos.logback;
org.xml;
javax.xml;
org.apache.xerces;
org.w3c
```





## 参考资料

[通过源码理解Java类加载机制与双亲委派模型](https://www.jianshu.com/p/67021213872a)

[再谈双亲委派模型与Flink的类加载策略](https://www.jianshu.com/p/bc7309b03407)

[Flink中的类加载机制](https://blog.csdn.net/chenxyz707/article/details/109043868)

[简介 FLINK 上传与加载用户依赖的策略](https://zhuanlan.zhihu.com/p/93374622)