---
layout:     post
title:      Zookeeper
subtitle:   
date:       2019-03-26
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - bigdata
    - Zookeeper
---


### 简介

**Zookeeper** 是一个高可用的分布式数据**管理**和**协调**框架，并且能够很好的保证分布式环境中数据的**一致性**。在越来越多的分布式系统（`Hadoop`、`HBase`、`Kafka`）中，`Zookeeper` 都作为核心组件使用。

#### Zookeeper 结构

![](https://vendanner.github.io/img/hadoop/Zookeeper1.png)


- Following：
	- 处理客户端**非事物**请求并向客户端返回结果
	- 将**事物**请求转发给`Leader`
	- 同步 `Leader`  状态
	- 选 `Leader` 参与投票            
- Leader：
	- **事物**请求**唯一**调度者和处理者，保证集群事物处理的**顺序性**
	- 集群内部各服务器的**调度者**
- Observer：

由上可知，`Following` 只处理**非事务**请求，而**事务**请求如：节点创建、节点删除、读写数据由 `Leader` 统一处理。


### 选举

集群刚启动的时候，无 `Following` 和`Leader` 之分，需要先选举出 `Leader` (票数多胜出)。

 - 每个节点发出一个投票，内容为(`myid`,`ZXID`) - `myid` 为节点ID，`ZXID` 是集群事物ID
 - 接收来自各节点的投票
 - 处理投票(先比较 `ZXID`，再比较 `myid`)
 - 统一投票(得票数超一半则为 `Leader`)
 - 更改节点状态：`Following`  or `Leader`

![](https://vendanner.github.io/img/hadoop/Zookeeper2.png)

值得一提的是，成为 `Leader` 条件是得票数出超半，所以存活节点需要满足以下公式：
 > live_node >= total_node/2+1，total_node 一般为奇数

接着来说说处理投票规则(先比较 `ZXID`，再比较 `myid`)。考虑这么一种情况，在运行期间 `Leader` 挂了，此时需要重新选举 `Leader`。

![](https://vendanner.github.io/img/hadoop/Zookeeper3.png)

之前的 `Leader` 挂了之后，有些 `ZXID` 处理状态没有同步到所有节点。此时先比较 `ZXID` 就很明智：发出 `ZXID` 高的投票当选 `Leader` 后，会先将 `ZXID`  **再次**向所有节点同步保证数据的一致性。


### 配置

#### 配置文件

配置文件在 `zookeeper-3.4.10/conf` 目录下，拷贝一份 `zoo_sample.cfg` 为 `zoo.cfg` 再修改。

	dataDir=/home/hadoop/software/zookeeper-3.4.10/data   // zookeeper 数据存放路径
	dataLogDir=/home/hadoop/software/zookeeper-3.4.10/logs  // zookeeper 日志存放路径
	// 节点配置，端口是固定的
	server.1=node00:2888:3888
	server.2=node01:2888:3888
	server.3=node02:2888:3888
	server.4=node03:2888:3888
	server.5=node04:2888:3888

#### 环境变量

	export ZOOKEEPER_HOME=/home/hadoop/software/zookeeper-3.4.10
	export PATH=\$JAVA_HOME/bin:\$PATH:$ZOOKEEPER_HOME/bin

#### 创建文件夹

	配置文件中我们指定两个 `data` 目录，要先创建
	`DataDir` 目录下创建 `myid` 文件来指定当前 `zookeeper`  节点ID
	/home/hadoop/software/zookeeper-3.4.10 目录下
	mkdir data
	mkdir logs
	cd data
	touch myid
	echo "1" > myid //指定节点编号为1

#### 启动

	zkServer.sh start //启动 zookeeper
	jps 查看是否运行 zookeeper
	[hadoop@node02 ~]$ jps
	28688 Jps
	28622 QuorumPeerMain    // 出现这个进程表示 zookeeper 在运行

注意： `zookeeper` 在运行只是单纯的表示程序在执行，并不能说明 `zookeeper` 集群已经启动；在启动过程中有可能会**出错**。` zkServer.sh start` 命令在哪个目录执行，就会在**当前目录**产生 `zookeeper.out` 文件，其中会记录 `zookeeper`  启动日志，报错信息可以在这里查看。

	zkServer.sh status // 查看 zookeeper 状态
	// 若 zookeeper 集群**正常运行**，Leader 显示
	Mode: leader
	// follower 显示
	Mode: follower

注意启动 `zookeeper`  的节点应符合之前描述 `live_node` ，不然无法正常运行。


### Znode

`zookeeper`  存储数据的单元称为 `Znode`，它的视图结构与 `Linux` 类似，但没有目录和文件概念。

![](https://vendanner.github.io/img/hadoop/Zookeeper4.png)

![](https://vendanner.github.io/img/hadoop/Zookeeper5.png)

值得注意的是**临时节点**不允许再有子节点。

`Znode` 内容如下：

	3453    // Znode 数据
	// 创建 znode 事务ID
	cZxid = 0x100000002   
	ctime = Sun Apr 07 15:19:42 CST 2019
	// 修改 znode 事务ID
	mZxid = 0x100000005
	mtime = Sun Apr 07 15:21:13 CST 2019
	// 用于添加或删除子节点的znode更改的事务ID。
	pZxid = 0x100000006
	cversion = 1
	dataVersion = 2 // 版本号，每修改一次加1，原始为0
	aclVersion = 0
	ephemeralOwner = 0x0
	dataLength = 4 // 数据长度
	numChildren = 1 // 子节点数


### 应用

分布式系统节点动态上下线

![](https://vendanner.github.io/img/hadoop/Zookeeper6.png)

- 服务器上线后，在 `Zookeeper` 创建临时节点表示新服务器上线；服务器下线后之前的临时节点自动**删除**
- 客户端会**监听**目录，当新增或者删除节点时代表服务器变化需要**更新**



## 参考资料：
[【分布式】Zookeeper应用场景](https://www.cnblogs.com/leesf456/p/6036548.html)
