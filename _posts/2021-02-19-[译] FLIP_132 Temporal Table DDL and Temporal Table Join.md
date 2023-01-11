---
layout:     post
title:      [译] FLIP_132 Temporal Table DDL and Temporal Table Join
subtitle:   
date:       2021-02-19
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - SQL
---

[FLIP-132 Temporal Table DDL and Temporal Table Join](https://cwiki.apache.org/confluence/display/FLINK/FLIP-132+Temporal+Table+DDL+and+Temporal+Table+Join)

>  当维表可以使用 changelog 的方式获取时，支持 eventtime 的 join
>
> org.apache.flink.table.planner.plan.nodes.exec.stream.StreamExecTemporalJoin
>
> org.apache.flink.table.runtime.operators.join.temporal.TemporalRowTimeJoinOperator
>
> 测试： org/apache/flink/streaming/connectors/kafka/table/KafkaTableITCase.java#testKafkaTemporalJoinChangelog

### 动机

 临时表是视图的概念，该视图可以**根据特定时间点**返回表的内容。临时表包含具有版本信息的表快照，它可以 binlog ，也可以是一张数据库表。为了关联维表，Flink 使用 DDL 定义临时表，并在外部数据源 Lookup 。为了能关联到表的历史版本的数据，Flink 使用临时表函数将外部变化的表定义成 View，然后去访问它。但是临时表函数只能使用 Table API/YAML 调用，对于 SQL 很不方便。如果 Flink 支持以 DDL 方式定时临时表，SQL 用户会很方便。binlog 包含表所有的变化，在 FLIP-95 之后 Flink 就可以解析binlog。将时态表函数作用于 binlog 上，就可以访问数据库表特定版本的数据。

### 公共接口

#### 时态表概念

- 时态表：时态表是随时间变化的表，称为 Flink 动态表，时态表中的行与一个或多个时间周期相关联，所有 Flink 表都是时态（动态）表。
- 版本：时态表可以分为一组版本化的表快照，表快照中的版本表示行的**生命周期**，有效期的开始时间和结束时间可以由用户分配。时态表可以根据表是否跟踪其历史版本分为版本表和规则表。
- 版本表：如果时态表中的行可以记录其历史更改信息并访问其历史记录版本，则我们将这种动态表称为版本表
- 常规表：对于常规表，动态表中的行只能跟踪其最新版本。lookup join 中的表只能跟踪其最新版本，因此它也是一个常规表

#### 时态表关联概念

- 时态表关联：时态表关联是指任意表与动态表的**一个版本**的数据连接
- **Event-time** 时态表关联：**Event-time** 时态表关联是输入表使用它的**事件时间**在时态表联接中找到**相应版本**的动态表。
- **Processing-time **时态表关联：**Processing-time **时态表关联是输入表使用其处理时间在时态表联接中查找**最新版本**的动态表

#### 主键语义

Flink 中的主键约束意味着表或视图的一列或一组列是**唯一**的，并且它们不包含 null。但 Flink不拥有数据，因此我们要支持的唯一模式是**不强制模式**，由**用户来确保查询强制执行键完整性**。Flink 将使用主键约束来进行优化。

changelog source 上的主键语义意味着具体化的changelogs（INSERT/UPDATE_BEFORE/UPDATE_AFTER/DELETE）在主键约束上是唯一。Flink 假设所有消息在主键上都是有序的，并将使用主键更新临时连接操作符中的物化状态作为优化。

主键语义意味着对于 Append Source 所有输入记录在主键相同只会出现一次。如果主键上有重复记录，则查询结果可能不正确。

#### Event Time 语义

事件时间是每个单独事件在其产生设备上发生的时间。这个时间通常在记录进入Flink之前嵌入到记录中，并且可以从每个记录中提取事件时间戳。在事件时间中，时间的进程取决于数据，而不是任何时钟。changelog 中的事件时间也保留了这个语义，changelog 的事件时间戳（INSERT，UPDATE，DELETE）可以从每条记录中提取出来，它意味着用提取的时间插入/更新/删除一条记录。例如

- +INSERT(key, Flink, 09:00:00)：这条记录的事件时间是 09:00:00，insert 操作时间没有意义
- -DELETE(Key, Flink, 09:00:00)： 这条记录的事件时间是 09:00:00，delete 操作时间没有意义

#### 有版本的表/视图

我们建议使用主键和事件时间来定义版本表/视图

- 主键是记录具有相同主键的记录的不同版本所必需的
- 事件时间是划分每个记录的**有效生命周期**所必需的

```sql
-- 版本表建表语句示例
CREATE TABLE versioned_rates (
    currency STRING,
    rate DECIMAL(38, 10),
    currency_time TIMESTAMP(3),
    WATERMARK FOR currency_time AS currency_time, -- event time  (currency_time)
    PRIMARY KEY(currency) NOT ENFORCED            -- primary key (currency)
) WITH (
...
)
```

#### 常规表/视图

很容易理解，如果动态表不是版本表，那么它肯定是常规表。除版本表外，任何表都可以是常规表

```sql
CREATE TABLE latest_rates (  -- No event time and No primary key
    currency STRING,
    rate DECIMAL(38, 10),
    currency_time TIMESTAMP(3)      
) WITH (
...
)
```

#### 如何关联时态表

时态表最常见的场景是在 correlate 中关联指定的版本数据，目前只有时态表 join 支持版本表，任何地方都支持常规表。我们建议使用“FOR SYSTEM_TIME AS OF”来触发访问**指定版本**的时态表。“FOR SYSTEM_TIME AS of<point in TIME 1>” 语法在 SQL:2011 支持，语义是指返回**特定时间点**的时态表的视图。

```sql
-- 订单事实表
CREATE TABLE orders (
    order_id STRING,
    currency STRING,
    amount INT,
    proctime as PROCTIME(),                 
    order_time TIMESTAMP(3),                
    WATERMARK FOR order_time AS order_time - INTERVAL '30' SECOND
) WITH (
)

-- Event-time 时态表 join
SELECT 
  o.oder_id,
   o.order_time,
  o.amount * r.rate AS amount,
  r.currency
FROM orders AS o, versioned_rates FOR SYSTEM_TIME AS OF o.order_time r
on o.currency = r.currency;

-- Processing-time 时态表 join
SELECT 
  o.oder_id,
  o.proctime,
  o.amount * r.rate AS amount,
  r.currency
FROM orders AS o, latest_rates FOR SYSTEM_TIME AS OF o.proctime r
on o.currency = r.currency

```

### 提议的变更

建议的DDL将只支持 in-blink planner，因为旧版规划器已弃用，将很快被删除。

#### 有版本表/视图 DDL

- 请记住，版本表是包含主键和事件时间属性的动态表。
- 对于 Source 上的版本表，版本表的版本按用户特定的事件时间和主键拆分
- 对于视图上的版本表，如果查询包含推断的主键和事件时间属性，则它也有效

```sql
-- 以下都是版本表/视图

-- changelog 流中的延迟删除事件将被忽略，因为DELETE事件中的事件时间戳通常小于水印。下面的changelog流来自数据库binglog，包含DELETE事件，这些事件中的事件时间戳通常小于水印
-- 如果changelog源包含删除事件（versioned_rates2），建议使用 changelog time 作为事件时间
CREATE TABLE versioned_rates1 (
    currency STRING,
    rate DECIMAL(38, 10),
    currency_time TIMESTAMP(3),
    WATERMARK FOR currency_time AS currency_time, -- event time  (currency_time)
    PRIMARY KEY(currency) NOT ENFORCED            -- primary key (currency)
) WITH (
  'format' = 'debezium-json'/'canal-json'         -- changelog source
)
-- 版本表“versioned_rates2”是从包含主键和事件时间（changelog time）的changelog流生成的，该表包含多个版本
-- 与“versioned_rates1”相比，versioned表“versioned_rates2”中的版本是按changelog时间（即binlog时间）划分的，后者更适合处理changelog的延迟删除事件
-- Changelog time 是系统中发生更改的时间（即数据库中DML执行的系统时间），DELETE event 中的事件时间将是理想情况下不小于水印的执行时间
-- 如果changelog源包含DELETE事件，建议使用 changelog_time 作为事件时间
CREATE TABLE versioned_rates2 (
    currency STRING,
    rate DECIMAL(38, 10),
    currency_time TIMESTAMP(3),
    changelog_time TIMESTAMP(3) as SYSTEM_METADATA("db_operation_time"),
    WATERMARK FOR changelog_time AS changelog_time, -- event time  (changelog_time)
    PRIMARY KEY(currency) NOT ENFORCED              -- primary key  (currency)
) WITH (
  'format' = 'debezium-json'/'canal-json'           -- changelog source
)
-- 版本表“versioned_rates3” 是从仅附加的流中生成的，该流包含主键和事件时间（currency_time）
-- 该表是一个版本表，但每个记录只包含一个版本
-- 因为append only流中的主键意味着记录不会在主键上重复，所以只有一个版本
CREATE TABLE versioned_rates3 (
    currency STRING,
    rate DECIMAL(38, 10),
    currency_time TIMESTAMP(3),
    WATERMARK FOR currency_time AS currency_time, -- event time
    PRIMARY KEY(currency) NOT ENFORCED            -- primary key
) WITH (
  'format' = 'json'                               -- append only source
)
-- 版本表“versioned_rates4”将仅追加的源转换为changelog流，它包含推断的主键并保留事件时间属性
-- 该表也是一个版本表，每个记录都包含多个版本
-- 如果仅附加源包含业务键上的重复记录，建议使用“versioned_rates4”视图定义版本表
CREATE TABLE rates (
    currency STRING,
    rate DECIMAL(38, 10),
    currency_time TIMESTAMP(3),
    WATERMARK FOR currency_time AS currency_time   -- event time
) WITH (
  'format' = 'json'                               -- append only source
)
-- 该视图返回带有推断主键（Currency）和事件时间（Currency_time）的有序changelog流
CREATE VIEW versioned_rates4 AS               
SELECT currency, rate, currency_time
  FROM (
      SELECT *,
      ROW_NUMBER() OVER (PARTITION BY currency ORDER BY currency_time DESC) AS rowNum
      FROM rates )
WHERE rowNum = 1;
```

当 event 中的事件时间戳小于水印时，changelog 流中的延迟事件将被忽略，而 DELETE/UPDATE_BEFORE事件通常有一个延迟时间戳如果用户使用行中的 timestamp 字段作为事件时间，则表示这些DELETE/UPDATE_BEFORE事件将被忽略。UPDATE_BEFORE event对于版本表是可选的，可以忽略，因为总是有一个跟在后面的UPDATE_AFTER event 可以更新旧记录。删除事件是必要的，用于删除旧数据，不可忽略。

changelog 流来自数据库 binglog，包含 DELETE event 是一个常见的用户案例，如果用户从行字段中分配了事件时间，比如上面的案例“versioned_rates1”，DELETE event 中的事件时间戳总是小于水印，在 Flink 中会被视为 `late event`，因为数据库中的 delete 事件几乎是在删除事件时间较小的旧数据，所以下表显示，当从行字段分配事件时间时，delete event 总是有一个较迟的事件时间戳（10:05:00），而不是数据库系统时间（11:00:00）。

| DB system time(changelog time) | DB operation T(pk, name,biz_ts)                              | Produced changeling using biz_ts as event time               | Produced changeling using changelog time as event time       |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **11:00:00**                   | delete from T where pk = 'key'                               | -DELETE         (key, Hello, **10:05:00**)                   | -DELETE         (key, Hello, 10:05:00, **11:00:00**)         |
| **10:05:00**                   | update T set name = 'Hello', biz_ts = '10:05:00' where pk = 'key' | -UPDATE_BEFORE (key, Flink, **09:00:00**)<br /><br />+UPDATE_AFTER  (key, Hello, **10:05:00**) | -UPDATE_BEFORE (key, Flink, 09:00:00, **10:05:00**)<br /><br />+UPDATE_AFTER  (key, Hello, 10:05:00, **10:05:00**) |
| **09:00:00**                   | Insert into T (key, Flink, 09:00:00)                         | +INSERT         (key, Flink, **09:00:00**)                   | +INSERT         (key, Flink, 09:00:00, **09:00:00**)         |

在这种情况下，建议使用 changelog time 作为事件时间，changelog time 是系统中发生更改的时间（即数据库中DML执行的系统时间），我们可以从 binglog 中提取 changelog time，这将在FLIP-107中实现。版本表将基于changelog时间构建，DELETE事件中的changelog时间（11:00:00）接近数据库系统时间（11:00:00），理想情况下不会小于水印，如上表所示。

> 上面这一段其实就想表达一个意思：delete event 的事件时间是无法从 event 内提取的，如果勉强用 event 中其他时间，大概率是会因为时间远小于 Watermark 而丢弃。如果包含 delete ，建议是用数据库操作 event 的时间即 binlog 时间。

主键不应该在只附加的流上定义如果流在主键上有重复的记录，这会破坏主键语义，并可能在 Flink 中产生错误的结果。

业务键上有重复记录的append only源上的版本表是一种常见的用户情况，用户希望**更新**源语义。我们建议使用重复数据删除查询来定义这种版本表，视图“versioned_rates4”中的查询是一个重复数据删除查询，它可以将 append only 流转换为 changelog 流，changelog 流包含主键上的 INSERT 和 UPDATE 事件，并保留事件时间。顺便说一句，此查询中的模式称为“重复数据消除查询”，并将在 planner 和 runtime 中针对重复数据消除节点进行优化。

这样一来，一个好处是我们不需要在可能包含重复记录的仅附加源中定义主键，主键语义变得清晰，另一个好处是上述视图的结果是一个 changelog 流，我们在概念上统一了时态表和 changelog 。另外，用户可以在视图的查询中进行过滤/投影/聚合/连接，极大地提高了灵活性。

#### 常规表/视图

如果动态表不是版本化的时态表，那么它就是常规表。常规表意味着表或视图**不能同时拥有主键和事件时间**。

```sql
CREATE TABLE latest_rates1 ( -- No event time and No primary key
    currency STRING,
    rate DECIMAL(38, 10) 
) WITH (
)

CREATE VIEW latest_rates2 AS -- No event time and No primary key
SELECT currency, rate 
FROM rates;                 

CREATE TABLE latest_rates3 ( -- No primary key
    currency STRING,
    rate DECIMAL(38, 10),
    currency_time TIMESTAMP(3),
    WATERMARK FOR currency_time AS currency_time, 
) WITH (
)

CREATE TABLE latest_rates4 ( -- No event time
    currency STRING,
    rate DECIMAL(38, 10),
    proctime AS PROCTIME(),
    PRIMARY KEY(currency) NOT ENFORCED 
) WITH (
)

CREATE VIEW latest_rates5 AS -- No event time and only has a inferred primary key
SELECT currency, rate, proctime
  FROM (
      SELECT *,
      ROW_NUMBER() OVER (PARTITION BY currency ORDER BY proctime DESC)
      AS rowNum FROM Rates)
WHERE rowNum = 1;
```

对于常规表，表只保留表的**最新版本**，它可以是除版本表以外的任何表或任何视图。

### 兼容性、弃用和迁移计划

#### 迁移计划

```scala
// Event-time temporal table join 示例

// The Temporal Table Function way (这种方式不建议，已弃用)
Table ratesHistory = tEnv.fromDataStream(ratesHistoryStream, $("currency"), $("rate"), $("order_time").rowtime());
tEnv.createTemporaryView("versioned_rates", ratesHistory);
TemporalTableFunction rates = ratesHistory.createTemporalTableFunction("order_time", "currency"); // <==== (1)
tEnv.registerFunction("rates", rates);     
tEnv.executeSql("SELECT
      o.oder_id,
      o.order_time,
      o.amount * r.rate AS amount,
      r.currency
    FROM
      Orders AS o,
      LATERAL TABLE(rates(o.order_time))AS r
    WHERE o.currency = r.currency");
```

建议用这种方式

```sql
-- The proposed Temporal Table DDL way, create a versioned table
CREATE VIEW versioned_rates AS
SELECT currency, rate, currency_time
FROM (
      SELECT *,
      ROW_NUMBER() OVER (PARTITION BY currency ORDER BY currency_time  DESC) AS rowNum
      FROM rates)
WHERE rowNum = 1;


SELECT 
  o.oder_id,
  o.order_time,
  o.amount * r.rate AS amount,
  r.currency
FROM orders AS o, versioned_rates FOR SYSTEM_TIME AS OF o.order_time r
on o.currency = r.currency;
```

