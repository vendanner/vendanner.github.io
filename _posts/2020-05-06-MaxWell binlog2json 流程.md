---
layout:     post
title:      MaxWell binlog2json 流程 
subtitle:   
date:       2020-05-06
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - MaxWell
    - bigdata
---

下载源码，执行如下命令编译

> mvn -e clean package -DskipTests=true

MaxWell 中，Binlog2json 流程是**单线程**(main thread)执行。整个执行流程如下所示

![](https://vendanner.github.io/img/maxwell/step.jpg)

了解完整的解析流程，我们就可以做一些二次开发：

- DDL 语句：业务表结构发生变化后，下游大数据如何感知？在 **SQL 解析**部分添加代码
- 时间序列：若业务的 lastupdate 字段和binlog postion 值都无法**验证** sql 语句的先后执行顺序，可在构造json 时添加 `index` 来保障 sql语句 的先后关系(单线程)。