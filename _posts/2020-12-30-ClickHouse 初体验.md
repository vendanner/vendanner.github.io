---
layout:     post
title:      ClickHouse 初体验
subtitle:   
date:       2020-12-30
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - ClickHouse
    - bigdata
---

现在有查明细和聚合需求，对比现有的 OLAP 引擎，我们选择 ClickHouse 来做。

ClickHouse 有很多的引擎，生产一般直接上 **MergeTree**，这种引擎支持**主键索引**、**数据分区**、**数据副本**、**数据采样**也支持 `Alter` 操作。我们选择的是 `ReplicatedReplacingMergeTree`，表示支持副本并可以删除重复数据。

### 安装

检查是否支持 `SSE 4.2`，否则 ClickHouse 无法运行

> grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"

离线[下载](https://packagecloud.io/Altinity/clickhouse) 

```shell
# 注意区分当前linux操作系统的 el 版本号，本机是 el7
clickhouse-client-20.8.3.18-1.el7.x86_64.rpm
clickhouse-common-static-20.8.3.18-1.el7.x86_64.rpm
clickhouse-server-20.8.3.18-1.el7.x86_64.rpm 
clickhouse-server-common-20.8.3.18-1.el7.x86_64.rpm
```

```shell
# 安装
rpm -ivh clickhouse-common-static-20.8.3.18-1.el7.x86_64.rpm 
rpm -ivh clickhouse-server-common-20.8.3.18-1.el7.x86_64.rpm 
rpm -ivh clickhouse-server-20.8.3.18-1.el7.x86_64.rpm
Create user clickhouse.clickhouse with datadir /var/lib/clickhouse  # 新建 clickhouse 用户
rpm -ivh clickhouse-client-20.8.3.18-1.el7.x86_64.rpm 
# 执行文件; 命令输入 clickhouse 后按 tab 可查看 clickhouse 相关命令
/etc/init.d/clickhouse-server
/usr/bin/clickhouse-client  
# 成功后，配置文件
/etc/clickhouse-server/
/etc/clickhouse-client/ 
# 日志文件
/var/log/clickhouse-server
```

修改默认的 `data` 文件路径

```shell
# /etc/clickhouse-server/config.xml
    <!-- Path to data directory, with trailing slash. -->
    <path>/var/lib/clickhouse/</path>
```

#### 启动

```shell
/etc/init.d/clickhouse-server start
# 启动成功后，ps 和 netstat 查看 clickhouse 是否正常
ps aux| grep clickhouse
netstat -nlp | grep clickhouse      # 监听三个端口
# 若 clickhouse 启动异常，查看日志
/var/log/clickhouse-server/clickhouse-server.err.log
/var/log/clickhouse-server/clickhouse-server.log
```

Client 访问

```shell
# 本机访问，若远程访问还需集群部署设置，下文介绍
clickhouse-client --host localhost --port 9000
```

#### 集群部署

另外三台相同部署，确保启动正常。在正式部署前先讲解**分片**和**副本**的概念

- 分片：类比传统数据库的分表，数据横行切分(每个表设置不同的分片数)
- 副本：分布式存储必有，ClickHouse 可以在**分片**层提供副本机制(在文件中设置)，数据同时向所有副本写入实现高可用，但这种方案会有 `bug`(不会检查副本一致性)；生产中推荐使用复制表(Replicated*MergeTree)来实现副本，这种方式由 ClickHouse 来保证数据副本的一致性。
- 每个 ClickHouse 实例只能提供一个分片

| 主机 |                    |                    |
| :--- | ------------------ | ------------------ |
| Cdh1 | 9000: 01分片01副本 | 9001: 04分片02副本 |
| Cdh2 | 9000: 02分片01副本 | 9001: 01分片02副本 |
| Cdh3 | 9000: 03分片01副本 | 9001: 02分片02副本 |
| Cdh4 | 9000: 04分片01副本 | 9001: 03分片02副本 |

```xml
<!-- /etc/clickhouse-server/metrika.xml -->
<yandex>
  <clickhouse_remote_servers>
    <!-- 集群名称 -->
    <cluster_4shards_2replicas>
      <shard>
        <!-- 数据存储占比权重 -->
        <weight>1</weight>
        <!-- 只写一个副本，靠 Replicated 引擎复制 -->
        <internal_replication>true</internal_replication>
        <replica>
          <host>cdh220.ops.com</host>
          <port>9000</port>
          <user>default</user>
          <password></password>
        </replica>
        <replica>
          <host>cdh222.ops.com</host>
          <port>9002</port>
          <user>default</user>
          <password></password>
        </replica>
      </shard>
      <shard>
        <!-- 数据存储占比权重 -->
        <weight>1</weight>
        <internal_replication>true</internal_replication>
        <replica>
          <host>cdh222.ops.com</host>
          <port>9000</port>
          <user>default</user>
          <password></password>
        </replica>
        <replica>
          <host>cdh223.ops.com</host>
          <port>9002</port>
          <user>default</user>
          <password></password>
        </replica>
      </shard>
      <shard>
        <!-- 数据存储占比权重 -->
        <weight>1</weight>
        <internal_replication>true</internal_replication>
        <replica>
          <host>cdh223.ops.com</host>
          <port>9000</port>
          <user>default</user>
          <password></password>
        </replica>
        <replica>
          <host>cdh224.ops.com</host>
          <port>9002</port>
          <user>default</user>
          <password></password>
        </replica>
      </shard>
      <shard>
        <!-- 数据存储占比权重 -->
        <weight>1</weight>
        <internal_replication>true</internal_replication>
        <replica>
          <host>cdh224.ops.com</host>
          <port>9000</port>
          <user>default</user>
          <password></password>
        </replica>
        <replica>
          <host>cdh220.ops.com</host>
          <port>9002</port>
          <user>default</user>
          <password></password>
        </replica>
      </shard>
    </cluster_4shards_2replicas>
  </clickhouse_remote_servers>
    
  <!-- zookeeper相关配置 -->
  <zookeeper-servers>
    <node index="1">
      <host>cdh222.ops.com</host>
      <port>2181</port>
    </node>
    <node index="2">
      <host>cdh223.ops.com</host>
      <port>2181</port>
    </node>
    <node index="3">
      <host>cdh224.ops.com</host>
      <port>2181</port>
    </node>
  </zookeeper-servers>

  <!-- 以下的配置根据节点各自具体配置，宏定义在创建表时使用-->
  <macros>
    <!-- 分片序号 -->
    <shard>01</shard>
    <!-- 副本名称 -->
    <replica>cdh220.ops.com</replica>
  </macros>
  
  <networks>
    <ip>::/0</ip>
  </networks>

  <!-- 数据压缩配置 -->  
  <clickhouse_compression>
    <case>
      <min_part_size>10000000000</min_part_size>
      <min_part_size_ratio>0.01</min_part_size_ratio>
      <method>lz4</method>
    </case>
  </clickhouse_compression>

</yandex>
```

每台机子启动两个 ClickHouse 实例，复制 config 文件为 config2.xml，并修改

```xml
<!-- config 修改内容 -->

<!-- 时区配置 -->
<timezone>Asia/Shanghai</timezone>

<!-- 以下内容同个机子上的 config 要单独配置 -->
<!-- 日志文件地址修改 -->
<logger>
  <log>/var/log/clickhouse-server/clickhouse-server.log</log>
  <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
<!-- 端口配置 config1 默认，下面是 config2 配置;netstat -nlp | grep LIST |grep 9000 查看端口占用情况 -->
  <http_port>8124</http_port>
  <tcp_port>9003</tcp_port>
  <interserver_http_port>9012</interserver_http_port>
  
<!-- 目录配置 -->
  <path>/data/clickhouse/data/</path>
  <tmp_path>/data/clickhouse/tmp/</tmp_path>
  <user_files_path>/data/clickhouse/user_files/</user_files_path>
  <format_schema_path>/data/clickhouse/format_schemas/</format_schema_path>
  
  <path>/data/clickhouse/data/</path>
  <tmp_path>/data/clickhouse2/tmp/</tmp_path>
  <user_files_path>/data/clickhouse2/user_files/</user_files_path>
  <format_schema_path>/data/clickhouse2/format_schemas/</format_schema_path>
  
<!-- 导入 metrika.xml 配置，文件名称要改 -->
<include_from>/etc/clickhouse-server/metrika.xml</include_from>
```

复制 `clickhouse-server` 为 ， `clickhouse-server2` 并修改

```bash
CLICKHOUSE_DATADIR=/data3/clickhouse/data
```

```shell

CLICKHOUSE_LOGDIR=/var/log/clickhouse-server2
CLICKHOUSE_DATADIR=/data4/clickhouse/data
# /etc/cron.d/clickhouse-server 复制一份
CLICKHOUSE_CRONFILE=/etc/cron.d/clickhouse-server2
CLICKHOUSE_CONFIG=$CLICKHOUSE_CONFDIR/config2.xml
CLICKHOUSE_PIDFILE="$CLICKHOUSE_PIDDIR/$PROGRAM-2.pid"
```

### 测试

生产上创建**复制表**(Replicate*MergeTree)，然后在此基础上创建**分布式表**；操作时推荐**写本地表、读分布式表**。

```sql
-- 增加 ON CLUSTER 'xxx' 描述会在集群所有实例上执行 sql 
-- 在 cluster_4shards_2replicas 集群的所有分片上创建 test 数据库
create database if not exists test on CLUSTER 'cluster_4shards_2replicas';

-- 创建 ReplicatedReplacingMergeTree 表,本地表通常后缀加 _local
-- /clickhouse/tables/{shard}/test/events_local, ZK中该表相关数据的存储路径,规范为 /clickhouse/tables/{shard}/[database_name]/[table_name]
-- {shard} 和 {replica} 是 metrika.xml 中 macros的值
-- TTL ts_date + INTERVAL 1 MONTH ； 数据生命周期 1 个月
CREATE TABLE IF NOT EXISTS test.events_local ON CLUSTER 'cluster_4shards_2replicas' (
  ts_date Date,     -- 时间到天
  user_id UInt64,
  state String,
  lastUpdateTime DateTime,      -- 时间到秒
  insertIntoCK DateTime DEFAULT now()
)
ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{shard}/test/events_local','{replica}')
PARTITION BY ts_date
ORDER BY (user_id)
TTL ts_date + INTERVAL 1 MONTH;

-- 在复制表的基础上创建分布式表，后缀名 _all
-- Distributed(集群标识符，本地表的数据库名称，本地表名称，数据分片函数)
CREATE TABLE IF NOT EXISTS test.events_all ON CLUSTER cluster_4shards_2replicas
as test.events_local
ENGINE = Distributed(cluster_4shards_2replicas,test,events_local,intHash64(user_id));

-- 分布式表中插入数据
-- 三条数据会根据 intHash64(user_id) 分发到不同分片；数据是先写到一个分布式表的实例中并缓存起来，再逐渐分发到各个分片上去，写放大
INSERT INTO TABLE events_all(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10001,'100','2021-01-05 10:11:01';
INSERT INTO TABLE events_all(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10002,'100','2021-01-05 10:11:01';
INSERT INTO TABLE events_all(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10003,'100','2021-01-05 10:11:01';

-- 本地表插入数据
-- 当前数据插入到执行 sql 的分片上
INSERT INTO TABLE events_local(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10004,'100','2021-01-05 10:11:01';
```

#### 视图

```sql
-- 插入相同 user_id 数据
INSERT INTO TABLE events_all(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10002,'200','2021-01-05 11:11:01';

-- 出现两条 10002 数据
SELECT * from events_all;

-- 手动触发合并
optimize table events_local final;
SELECT * from events_all;       -- 只有一条 10002 数据
```

`Replacing` 合并树引擎可以根据**主键去重**，但有很多限制：

- **触发合并**(默认1天)之后才去重，只能保证最终数据一致性
- 无法对分布式表合并，所以**无法跨分片去重** 
- 只对**同一分区内**的数据去重，无法跨区，在写入数据时一定要注意
- **ORDER BY的作用，** 负责分区内**数据排序**；**PRIMARY KEY**的作用， 负责一级**索引**生成；**Merge**的逻辑， 分区内数据排序后，找到相邻的数据，做特殊处理

在合并后才能实现主键去重显然是无法满足需求，这里尝试增加一层**视图**来实现。

```sql
-- 创建视图 
CREATE VIEW IF NOT EXISTS test.events_view
ON CLUSTER cluster_4shards_2replicas
as SELECT 
argMax(ts_date, insertIntoCK) as ts_date,
user_id,
argMax(state, insertIntoCK) as state,
argMax(lastUpdateTime, insertIntoCK) as lastUpdateTime,
max(insertIntoCK) as insertTime
from test.events_local
group by user_id;

-- 在视图的基础上创建 分布式物化视图
CREATE TABLE IF NOT EXISTS test.events_view_all
ON CLUSTER cluster_4shards_2replicas
AS test.events_view
ENGINE = Distributed(cluster_4shards_2replicas,test,events_view,intHash64(user_id));

-- 分布式表插入数据
 INSERT INTO TABLE events_all(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10006,'100','2021-01-05 10:11:01';
 INSERT INTO TABLE events_all(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10001,'300','2021-01-05 12:11:01';
 INSERT INTO TABLE events_all(ts_date,user_id,state,lastUpdateTime) select '2021-01-05',10001,'600','2021-01-05 14:11:01';
 
 -- 无重复数据
 select * from test.events_view_all
```

### 运维与实践

#### 实践

- 建议指定`use_minimalistic_part_header_in_zookeeper = 1`设置项，能够显著压缩表元数据在ZooKeeper中的存储。
- 写本地表，读分布式表
- **谓词下推**弱，最好手动添加
- 在单词基数少于10万的情况下，使用 `LowCardinality(String)` 代替 `String`，可以提高查询效率和减少存储空间但写入速度有一定的下降
- 优先使用 `in` 而不是 `join`
- 分布式表的 `in` 、`join` 操作加 `GLOBAL` 修饰符，会增加中间缓存防止读放大
- 清理分布式DDL 日志记录

```xml
<!-- 每执行一条分布式ddl，都会在 zookeeper /clickhouse/task_queue/ddl 目录 新建 znode -->
<!-- config.xml -->
<distributed_ddl>
  <!-- Path in ZooKeeper to queue with DDL queries -->
  <path>/clickhouse/task_queue/ddl</path>
  <!-- 默认60s，周期性检查ddl 是否可删除的间隔-->
  <cleanup_delay_period>60</cleanup_delay_period>
   <!-- 默认7天，ddl 文件可保留最大时长-->
  <task_max_lifetime>86400</task_max_lifetime>
  <!-- 默认1000，ddl 文件可保留的最大文件数-->
  <max_tasks_in_queue>200</max_tasks_in_queue>
</distributed_ddl>
```

#### 监控

```shell
# config.xml 放开 prometheus 注释
    <prometheus>
        <endpoint>/metrics</endpoint>
        <port>9363</port>

        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
        <status_info>true</status_info>
    </prometheus>
```

安装 `prometheus`

在 prometheus.yml 中增加 CH 的 Endpoint 地址:

```yaml
- job_name: 'clickhouse'
    static_configs:
    - targets: ['cdh220.ops.com:9363','cdh220.ops.com:9364']
```

在 `grafana` 添加 `prometheus` 数据源，可视化展示。





## 参考资料

[Clickhouse集群部署](https://www.cnblogs.com/jiashengmei/p/11991243.html)

[ClickHouse集群搭建（二）](https://segmentfault.com/a/1190000038329831)

[ClickHouse高可用集群的安装与部署](https://www.jianshu.com/p/78271ba9969b)

[利用复制表ReplicatedMergeTree实现ClickHouse高可用](http://cxy7.com/articles/2019/06/07/1559910377679.html)

[如何选择ClickHouse表引擎](https://help.aliyun.com/document_detail/156340.html)

[ClickHouse各种MergeTree的关系与作用](https://mp.weixin.qq.com/s?__biz=MzA4MDIwNTY4MQ==&mid=2247483804&idx=1&sn=b304f7f88d064cc08f87fa5eaafec0b7&chksm=9fa68382a8d10a9440d3ce2a92a04c4a74aeda2d959049f04f1a414c1fb8034b97d9f7243c21&scene=21#wechat_redirect)

[ClickHouse Better Practices](https://www.jianshu.com/p/363d734bdc03)

[尝鲜ClickHouse原生EXPLAIN查询功能](https://cloud.tencent.com/developer/article/1662230)

[ClickHouse常用的监控指标有哪些？](https://my.oschina.net/u/4579603/blog/4864980)

[ClickHouse 使用指南](https://blog.alexanderliu.top/posts/clickhouse-user-guide.html)

[CentOS7安装部署Prometheus+Grafana](https://www.jianshu.com/p/967cb76cd5ca)

[百分点 ClickHouse 项目实践](https://www.infoq.cn/article/Ah9eVLoPzTLV2a9vGP93)

[ClickHouse学习系列之三【配置文件说明】](https://www.iscxz.com/3472/)、

[在mac上用clion编译调试clickhouse流程](https://blog.csdn.net/xueweiThanos/article/details/112512545)