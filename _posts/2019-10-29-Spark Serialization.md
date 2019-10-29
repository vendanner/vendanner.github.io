---
layout:     post
title:      Spark Serialization
subtitle:   
date:       2019-10-29
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spark
    - Serialization
    - bigdata
---

>   https://spark.apache.org/docs/latest/tuning.html#data-serialization 

Spark 需要在 Executor 和 Driver 之间传输数据，必然是涉及到**序列化**的问题。序列化后数据的大小和反序列执行的速度肯定对整个 Spark 作业有很大的影响。Spark 提供两种序列化方式：

- java 序列化：spark **默认**使用 `  Java’s ObjectOutputStream`，但也可以继承[java.io.Serializable](https://docs.oracle.com/javase/8/docs/api/java/io/Serializable.html) 来自定义并使用 [java.io.Externalizable](https://docs.oracle.com/javase/8/docs/api/java/io/Externalizable.html) 使序列化的数据结构更紧密。**Java 序列化可以做到很灵活，但其执行速度慢且占据空间大**。
- Kryo 序列化：相比于 Java 序列化，**执行速度更快占据空间更小**，但需要使用  *register*  你需要的类才能达到更优的性能。在 2.0.0 开始，在处理简单类型时，spark 内部已使用的是 `Kryo` 。

### 测试

在讲解两种序列化的优缺点之后，我们写代码来测试看看

#### Java

```scala
seriRdd.persist()
```

查看 `seriRdd` 占用内存

![](https://vendanner.github.io/img/Spark/java_cache.png)

序列化存储

```scala
seriRdd.persist(StorageLevel.MEMORY_ONLY_SER)
```

![](https://vendanner.github.io/img/Spark/java_seri_cache.png)

很显然，序列化后占用内存减少了(但反序列化是要时间的)

### `Kryo`

```scala
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
seriRdd.persist(StorageLevel.MEMORY_ONLY_SER)
```

![](https://vendanner.github.io/img/Spark/kryo_ser_noregister.png)

虽然使用了 `KryoSerializer`，但**没有注册**，占用的空间比 java 序列化还大。那来看看注册之后的对比

```scala
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
conf.registerKryoClasses(Array(classOf[TestSerialization]))
seriRdd.persist(StorageLevel.MEMORY_ONLY_SER)
```

![](https://vendanner.github.io/img/Spark/kryo_seri_register.png)

当注册之后，内存占用还是下降蛮多。**切记使用 `Kryo` 一定要注册**。

总结下占用内存，从大到小排序

- 没序列化 `234 M`
- 使用 Kryo 序列化，但没注册 `156.8 M`
- 使用 Java 序列化  `67.4 M`
- 使用 Kryo 序列化并注册 `32.3 M`