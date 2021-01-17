---

layout:     post
title:      Flink 之 Retract
subtitle:   
date:       2020-06-02
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
---

实时计算过程中**上游数据**可能会有修改，此时下游数据也需要做相应的**更正**，这种功能在 Flink 中称为 `Retract`。

``` scala
object RetractStreamApp {
  def main(args: Array[String]): Unit = {
    val timeStamp = misc.getLongTime("2020-03-05 00:00:00")

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val bsSettings = EnvironmentSettings.newInstance.useBlinkPlanner.inStreamingMode.build
    val tabEnv = StreamTableEnvironment.create(env,bsSettings)
    val kafkaConfig = new Properties()
    kafkaConfig.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
    kafkaConfig.put(ConsumerConfig.GROUP_ID_CONFIG, "test")
    val consumer = new FlinkKafkaConsumer[String]("wc", new SimpleStringSchema, kafkaConfig)
      .setStartFromTimestamp(timeStamp)

    val ds = env.addSource(consumer).map((_, 1))
    tabEnv.createTemporaryView("wcTable", ds,'word,'cnt)
    val rsTable = tabEnv.sqlQuery("select word,sum(cnt) from wcTable group by word")
    // toRetract
    val result: scala.DataStream[(Boolean, Row)] = tabEnv.toRetractStream(rsTable)
    result.print()
    /* 撤回：kafka 数据 b,a,b,a,e
      false 就是需要撤回的数据
      最终结果是 (e,1),(a,2),(b,2)
      (true,e,1)
      (true,b,1)
      (true,a,1)
      (false,a,1)
      (true,a,2)
      (false,b,1)
      (true,b,2)
     */
    env.execute()
  }
}
```

这是一个简单 WordCount 例子，不同之处在于使用 SQL 来实现。代码的核心在于 `toRetractStream`:**流表**转化为可撤回流。看输出展示：

- 三个 `true` 表示插入三条数据
- (false,a,1) 后紧跟着 (true,a,2) 表示需要**撤回** (a,1)，然后**新插入** (a,2)
- (false,b,1) 后紧跟着 (true,b,2) 表示需要**撤回** (b,1)，然后**新插入** (b,2)

> 注意：当某个 Key  没有值时，Flink 是不会输出 (false, key, 0)

### 用途

以参考资料一中菜鸟物流为例：同一订单中，物流公司修改的情况下，就需要对下游的数据先做撤回然后新增的操作。

```sql
-- TT source_table 数据如下：
order_id      tms_company
0001           中通
0002           中通
0003           圆通

-- SQL代码
create view dwd_table as 
select
    order_id,
    StringLast(tms_company)
from source_table
group by order_id;

create view dws_table as 
select 
    tms_company,
    count(distinct order_id) as order_cnt
from dwd_table 
group by tms_company


-- 此时结果为：
tms_company  order_cnt
中通          2
圆通          1

-- 之后又来了一条新数据 0001的订单 配送公司改成 圆通了。
-- 这时，第一层group by的会先向下游发送一条 (0001,中通）的撤回消息，
-- 第二层group by节点收到撤回消息后，会将这个节点 中通对应的 value减少1，并更新到结果表中；
-- 然后第一层的分桶统计逻辑向下游正常发送(0001,圆通）的正向消息，
-- 更新了圆通物流对应的订单数目，达到了最初的汇总目的。

order_id      tms_company
0001           中通
0002           中通
0003           圆通
0001           圆通

-- 写入ADS结果会是（满足需求）
tms_company  order_cnt
中通          1
圆通          2
```





## 参考资料

[Flink SQL 功能解密系列 —— 流计算“撤回(Retraction)”案例分析](https://yq.aliyun.com/articles/457392)

[应用案例 \| Blink 有何特别之处？菜鸟供应链场景最佳实践](应用案例 | Blink 有何特别之处？菜鸟供应链场景最佳实践)

[Flink实战系列之自定义RetractStreamTableSink](https://mp.weixin.qq.com/s?__biz=MzU5MTc1NDUyOA==&mid=2247483877&idx=1&sn=c722beb68ae27e3d1ae757c68a6842cc&chksm=fe2b65aac95cecbc278412a50495fc101d7cfecdbe15d7f6a3e04a5dc184dba87a983789949b&token=1090913763&lang=zh_CN#rd)

[【Flink】流-表概念](https://www.cnblogs.com/leesf456/p/8027772.html)