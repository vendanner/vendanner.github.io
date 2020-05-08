---
layout:     post
title:      Spark SQL 之 Logical Plan
subtitle:   
date:       2020-03-31
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark
    - SQL
---

![](https://vendanner.github.io/img/Spark/catalyst_frontend.png)

上图展示了 `Logical Plan` 在整个 **SQL** 解析中所在的位置，并阐述 `Logical Plan` 经历的三个步骤

- **Unresolved Logical Plan**：由 `SparkSqlParser` 将语法树/DF/DataSet 转换成未解析的逻辑算子树，**不包含数据信息与列信息**
- **Logical Plan**：`Analyzer` 将一系列规则作用在未解析的逻辑树，对树节点**绑定各种数据信息**，生成解析后的逻辑树
- **Optimized Logical Plan**：`Optimeizer` 将**优化规则**作用到上一步的逻辑树，确保结果正确下**改写低效的结构**，生成优化后的逻辑算子树 

`Logical Plan` 主要记录了该逻辑节点处理逻辑相关的**属性**，包括**输入输出**、**约束条件**、**算子逻辑**和**统计信息**等。

### **Unresolved Logical Plan**

#### SparkSqlParser

![](https://vendanner.github.io/img/Spark/sparkSqlParser.jpg)

- ParserInterface：面向用户的接口，包含 SQL语句，表达式，数据标识符的解析方法
- AbstractSqlParser：实现 ParserInterface 并包含 AstBuilder
- AstBuilder：继承 SqlBaseBaseVisitor，生成语法抽象树
- SparkSqlAst-Builder：继承 AstBuilder，并增加 DDL 操作，在 SparkSqlParser 中调用
- SparkSqlParser：继承 AbstractSqlParser，用于外部调用



### **Logical Plan**



### **Optimized Logical Plan**





## 参考资料

Spark SQL 内核剖析