---
layout:     post
title:      FlumeToHDFS 故障案例 
subtitle:   
date:       2020-05-28
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flume
    - HDFS
    - 故障
    - bigdata
---

Flume 落盘到 HDFS 是在大数据非常普遍的场景，在生产中我们遇到个问题，特此记录。

### 现象

Flume 采集数据落到 Hive **小时分区**目录，Hive `T+1` ETL。出故障那天发现 Hive 表数据比业务表少个上万条记录，然后重新 ETL 加工后数据找回。这说明数据本来就是在 Hive 目录下，只是在 T+1 加工数据时没参与，而手动加工时缺失的数据参与 ETL 了。

### 分析

如上所述的现象，很容易让人联想到在 T+1 ETL 时有个文件被**其他占用**了导致没被加载进 Hive 表。查找Hive 目录发现确实有个文件在**24号**分区下，但最后的修改时间是**25号**。后来查找 Flume 日志发现这个文件一直**没有关闭**，导致无法加载进 Hive。

### 解决

知道是 Flume 占用文件的问题，就有了大致的解决方案：扫描 Hive 分区下被**其他应用**占据的文件，然后让其主动断开连接：

- hdfs fsck / -openforwrite | egrep -v '^\.+$' | egrep "MISSING|OPENFORWRITE" | grep -o "/[^ ]*" ：找到被占用的文件
- hdfs debug recoverLease -path 文件位置 -retries 重试次数：主动断开连接

### 原理

```tex
在HDFS中可能同时有多个客户端在同一时刻写文件，如果不进行控制的话，有可能多个客户端会并发的写一个文件，所以需要进行控制，一般的想法是用一个互斥锁，在某一时刻只有一个客户端进行写操作，但是在分布式系统中有如下问题：
1.每次写文件前，客户端需要向master获取锁情况，他们之间的网络通讯太频繁。
2.当某个客户端获取锁之后和master失去联系，这个锁一直被该客户端占据，master和其他客户端不能获得锁，后续操作中断。

在 HDFS 中使用了租约解决上面的问题：
1.当写一个文件时，客户端向 NameNode 请求一个租约，租约有个时间期限，在时间期限内客户端可以写租约中管理的文件，一个文件只可能在一个租约内，所以只可能有一个客户端写。
2.在租约的有效时间内，客户端不需要向 NameNode 询问是否有写文件的权限，客户端会一直持有，当客户端一直正常的时候，客户端在租约过期的时候会续约。
3.当客户端在持有租约期间如果发生异常，和 NameNode 失去联系，在租约期满以后 NameNode 会发现客户端异常，新的租约会赋给其他正常的客户端，当异常客户端已经写了一部分数据，HDFS为了分辨这些无用的数据，每次写的时候会增加版本号，异常客户端写的数据版本号过低，可以安全的删除掉。
```

简而言之，当 Flume 出故障一直没有去关闭文件时，由于**租约**规则 Hive 无法加载当前文件**全部内容**(修复文件后会发现文件比之前大很多)。

`recoverLease` 恢复租约，**释放文件之前的租约，Close 文件，报告 NameNode**。





## 参考资料

[HDFS的recoverLease和recoverBlock的过程分析](https://www.xuebuyuan.com/699824.html)

[Flink任务写hdfs文件卡在openforwrite状态](https://www.cnblogs.com/AloneAli/p/10840956.html)

[Hadoop浅解HDFS租约处理](https://blog.csdn.net/it_dx/article/details/57573534)