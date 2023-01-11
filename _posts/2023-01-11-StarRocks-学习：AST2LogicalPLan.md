---
layout:     post
title:      StarRocks 学习：AST2LogicalPLan
subtitle:
date:       2023-01-11
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - StarRocks
    - OLAP
---

> 从 SQL 文本到分布式物理执行计划, 在 StarRocks 中，需要经过以下 5 个步骤:
>
> 1、SQL Parse： 将 SQL 文本转换成一个 AST（抽象语法树）
> 2、SQL Analyze：基于 AST 进行语法和语义分析
> 3、SQL Logical Plan： 将 AST 转换成逻辑计划
> 4、SQL Optimize：基于关系代数、统计信息、Cost 模型，对逻辑计划进行重写、转换，选择出 Cost “最低” 的物理执行计划
> 5、生成 Plan Fragment：将 Optimizer 选择的物理执行计划转换为 BE 可以直接执行的 Plan Fragment

本文分析的是步骤三：在StarRocks 中如何将AST转换成LogicalPlan，供后续优化器优化。

StarRocks 源码中有丰富的测试案例，选择案例中的一条SQL来分析：

- 源码目录：com.starrocks.sql.analyzer.AnalyzeSubqueryTest
- SQL：select k from (select v1 as k from t0) a

## AST

本例中的SQL 经Analyze 后生成的AST，如下所示
![](https://vendanner.github.io/img/StarRocks/AST.png)

上图对应下面几个行为

- `TableRelation` : 包含表t0 的基本信息
- `SelectRelation`：
  - **select 内容从哪里来**：relation => TableRelation
  - **select 哪些字段**：selectList
- `SubqueryRelation`: 
  - 别名： alias
  - query： select
- `SelectRelation`: 
  - **select 内容从哪里来**：relation => SubqueryRelation
  - **select 哪些字段**：selectList

通过AST，大致了解本SQL要做哪些事情

1. 获取表数据
2. 表中有三列，但只需要v1 这列即可
3. select 后得到一个子查询，并命名为 "a"
4. 从表"a" 中取 "k" 这列

AST 到这里为止，下面看看LogicalPlan。



## LogicalPlan





大声点

## 参考资料

[一条查询SQL的StarRocks之旅](https://zhuanlan.zhihu.com/p/550520456)