---
layout:     post
title:      Spark shuffle 
subtitle:   
date:       2019-10-28
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spark
    - shuffle
    - bigdata
---

>  `Spark shuffle` 的概念与 `hadoop shuffle` 不同

在 `Hadoop` 中，`shuffle` 是发送在 **map -> reduce** 之间的数据传输，而在 Spark 中 shuffle 是发生在 **窄依赖 -> 宽依赖** 之间。那窄依赖、宽依赖是如何定义？

- 窄依赖：`父 RDD` 的每个分区只被**一个** `子 RDD` 的分区使用
- 宽依赖：`父 RDD` 的每个分区有可能被**多个** `子 RDD` 的分区使用

![](https://vendanner.github.io/img/Spark/shuffle_define.png)

确认好什么时候后会发送 shuffle 后，来看看 Spark shuffle  的演变之路

### `HashShuffle`

Spark 最初的版本就是 `HashShuffle`，勉强能用但有很多缺点。在 shuffle 会产生很多的**小文件**(shuffle 时若内存放不下会落到磁盘文件，而默认shuffle 内存大小是 `32k`，` spark.shuffle.file.buffer ` 参数 )：每个 task 根据下游的reduce 任务个数产生一个小文件，故小文件个数为 `n mapTask` * `m reduceTask` ；在实际生产在 n、m基数都是很大的，不免会产生**海量的小文件**可能导致 `OOM`。

![](https://vendanner.github.io/img/Spark/HashShuffle.png)

### `Consolidated HashShuffle`

`HashShuffle` 会产生这么多小文件，`Consolidated HashShuffle` 就是对它的改进。`Consolidated HashShuffle`  下，同个`Core`上执行的 task 会**共用** shuffle 文件即一个 `Core` 上不管有多少个 task 都只会生成 `m reduceTask` 个 shuffle 文件，对比于`HashShuffle` 是少了很多

![](https://vendanner.github.io/img/Spark/Consolidated HashShuffle.png)

### `SortShuffle`

上述两种方式产生的小文件个数都是两个数的**乘积**，若两个数的基数不小那难免还是会产生大量的小文件。在 `Spark  1.1.0`  引进`SortShuffle` 就很好的解决了难题：shuffle 文件个数与 `m reduceTask`无关，只与任务的 `Exceutor 个数`相关

![](https://vendanner.github.io/img/Spark/SortHash.png)

如上图所示，每个 `mapTask` 只产生2个文件：`data`、`index`

### 总结

![](https://vendanner.github.io/img/Spark/all_shuffle.png)

shuffle 文件数：`M = mapTask`，`C = 任务的 core 个数`，`R = reduceTask`

-  `basic hashshuffle` : `M * R`
- `Consolidated HashShuffle`：`C * R`
- `SortShuffle`：`2 * M`

