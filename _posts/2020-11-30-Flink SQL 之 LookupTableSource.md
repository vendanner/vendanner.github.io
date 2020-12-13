---
layout:     post
title:      Flink SQL 之 LookupTableSource
subtitle:   
date:       2020-11-30
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - bigdata
    - SQL
---

Flink 1.11

数仓建设的雪花模型：事实表 + 维表，维表是对事实表中某列数据的补全(商品id 补全商品名称、类型等)。维表是动态表，表里所存储的数据有可能不变，也有可能定时更新，但是更新频率不是很频繁。在实际生产中，维表一般是通过离线加工后存储。既然维表数据会有更新，在实时处理过程也是需要更新维表。Flink 的 Operator 层，可以通过以下操作更新维表：

- **预加载维表**：数据量不大情况下，open 方法打开连接然后新建线程**定时**加载维表
- **广播维表**：数据量不大情况下，维表数据当作封装成广播流，分发到其他流
- **AsyncIO**：一般和 Cache 搭配使用，**异步**请求维表数据，并缓存到 Cache

Flink SQL 层，框架已经封装好 **LookupTableSource** 接口(DynamicTable)，只需实现即可。1.11 版本已支持 JDBC、HBase 维表。下面以 JDBC 为例阐述 SQL 层维表的实现。

### LookupTableSource

```java
// org.apache.flink.table.connector.source.LookupTableSource
// 通过一个或多个 key 查询外部存储中数据
// 实际运行时，会通过 getLookupRuntimeProvider(LookupContext) 调用，而 key 保存在LookupContext.getKeys()
public interface LookupTableSource extends DynamicTableSource {
  // 读取数据的实现，实际就是 TableFunctin 底层去干活的，根据是否异步可分为：
  //   TableFunctionProvider
  //   AsyncTableFunctionProvider
  LookupRuntimeProvider getLookupRuntimeProvider(LookupContext context);
  interface LookupContext extends DynamicTableSource.Context {
    // 返回 keys 的 index
    // ROW < i INT, s STRING, r ROW < i2 INT, s2 STRING > >
    // when i and s2，返回 return [[0], [2, 1]]
    int[][] getKeys();
  }
  // TableFunctionProvider or AsyncTableFunctionProvider
  interface LookupRuntimeProvider 
}  
```

1.11 之前的版本

```java
public interface LookupableTableSource<T> extends TableSource<T> {
	TableFunction<T> getLookupFunction(String[] lookupKeys);
	AsyncTableFunction<T> getAsyncLookupFunction(String[] lookupKeys);
	boolean isAsyncEnabled();
}
```

1.11 版本更简单，LookupTableSource 只返回 LookupRuntimeProvider，而不是根据是否异步来调用不同的方法。

```java
// org.apache.flink.connector.jdbc.table.JdbcDynamicTableSource
// 注意还是实现了 SupportsProjectionPushDown(只获取使用到的字段，而不是 select *) 
public class JdbcDynamicTableSource implements ScanTableSource, LookupTableSource, SupportsProjectionPushDown {
  
  // 实现 LookupTableSource 接口
  public LookupRuntimeProvider getLookupRuntimeProvider(LookupContext context) {
  // context.getKeys() look up keys
  String[] keyNames = new String[context.getKeys().length];
  for (int i = 0; i < keyNames.length; i++) {
    int[] innerKeyArr = context.getKeys()[i];
    // keyNames 是 look up 列名
    keyNames[i] = physicalSchema.getFieldNames()[innerKeyArr[0]];
  }
  final RowType rowType = (RowType) physicalSchema.toRowDataType().getLogicalType();
  // getLookupRuntimeProvider 返回值确定了是否异步 look up
  return TableFunctionProvider.of(new JdbcRowDataLookupFunction(
    options,                             // 连接信息
    lookupOptions,                       // Cache 配置
    physicalSchema.getFieldNames(),      // 列
    physicalSchema.getFieldDataTypes(),  // 列的类型
    keyNames,                            // look up 列名
    rowType));
  }
}
```

### JdbcRowDataLookupFunction

JdbcRowDataLookupFunction 本质是 **TableFunction**。

> TableFunction最核心的就是eval方法，在这个方法里，做的主要的工作就是通过传进来的多个keys拼接成sql去来查询数据，首先查询的是缓存，缓存有数据就直接返回，缓存没有的话再去查询数据库，然后再将查询的结果返回并放入缓存，下次查询的时候直接查询缓存。

``` java
// org.apache.flink.connector.jdbc.table.JdbcRowDataLookupFunction
public class JdbcRowDataLookupFunction extends TableFunction<RowData> {
  public JdbcRowDataLookupFunction(
    JdbcOptions options,
    JdbcLookupOptions lookupOptions,
    String[] fieldNames,
    DataType[] fieldTypes,
    String[] keyNames,
    RowType rowType) {
  ...
  List<String> nameList = Arrays.asList(fieldNames);
  this.keyTypes = Arrays.stream(keyNames)
    .map(s -> {
      checkArgument(nameList.contains(s),
        "keyName %s can't find in fieldNames %s.", s, nameList);
      return fieldTypes[nameList.indexOf(s)];
    })
    .toArray(DataType[]::new);
  // TTL
  this.cacheMaxSize = lookupOptions.getCacheMaxSize();
  this.cacheExpireMs = lookupOptions.getCacheExpireMs();
  this.maxRetryTimes = lookupOptions.getMaxRetryTimes();
  // 根据 look up key 拼接 查询 sql
  // select fieldNames from options.getTableName where keyNames
  // open 函数中，使用 query 得到 statement 
  this.query = options.getDialect().getSelectFromStatement(
    options.getTableName(), fieldNames, keyNames);
  this.jdbcDialect = JdbcDialects.get(dbURL)
    .orElseThrow(() -> new UnsupportedOperationException(String.format("Unknown dbUrl:%s", dbURL)));
  this.jdbcRowConverter = jdbcDialect.getRowConverter(rowType);
  this.lookupKeyRowConverter = jdbcDialect.getRowConverter(RowType.of(Arrays.stream(keyTypes).map(DataType::getLogicalType).toArray(LogicalType[]::new)));
 }
  // 
  public void eval(Object... keys) {
    // 封装 keys
    RowData keyRow = GenericRowData.of(keys);
    if (cache != null) {
      // 首先从 Cache 中获取
      List<RowData> cachedRows = cache.getIfPresent(keyRow);
      if (cachedRows != null) {
        for (RowData cachedRow : cachedRows) {
          collect(cachedRow);
        }
        return;
      }
    }

    for (int retry = 1; retry <= maxRetryTimes; retry++) {
      try {
        statement.clearParameters();
        // 给 lookup key 设置 当前行查询的 key 值
        // getSelectFromStatement 生产 query 语句时，where 值是用 "?" 来代替的
        // 只有在真正运行时，"?" 才会被替代 
        statement = lookupKeyRowConverter.toExternal(keyRow, statement);
        try (ResultSet resultSet = statement.executeQuery()) {
          if (cache == null) {
            while (resultSet.next()) {
              collect(jdbcRowConverter.toInternal(resultSet));
            }
          } else {
            ArrayList<RowData> rows = new ArrayList<>();
            while (resultSet.next()) {
              // 返回值封装成 flink 内部数据类型 RowData
              RowData row = jdbcRowConverter.toInternal(resultSet);
              rows.add(row);
              collect(row);
            }
            rows.trimToSize();
            // Cache 缓存
            cache.put(keyRow, rows);
          }
        }
        break;
      } catch (SQLException e) {
        // 异常处理
        LOG.error(String.format("JDBC executeBatch error, retry times = %d", retry), e);
        if (retry >= maxRetryTimes) {
          throw new RuntimeException("Execution of JDBC statement failed.", e);
        }

        try {
          if (!dbConn.isValid(CONNECTION_CHECK_TIMEOUT_SECONDS)) 
            // timeout 重连，1.11 版本之前虽然会重试，但 connection 是同一个
            statement.close();
            dbConn.close();
            establishConnectionAndStatement();
          }
        } catch (SQLException | ClassNotFoundException excpetion) {
          LOG.error("JDBC connection is not valid, and reestablish connection failed", excpetion);
          throw new RuntimeException("Reestablish JDBC connection failed", excpetion);
        }

        try {
          Thread.sleep(1000 * retry);
        } catch (InterruptedException e1) {
          throw new RuntimeException(e1);
        }
      }
    }
  }
}
```







## 参考资料

[详解flink中Look up维表的使用](https://cloud.tencent.com/developer/article/1697903)

[Flink流与维表的关联](https://liurio.github.io/2020/03/28/Flink%E6%B5%81%E4%B8%8E%E7%BB%B4%E8%A1%A8%E7%9A%84%E5%85%B3%E8%81%94/)

[Flink Sql教程（7）](https://blog.csdn.net/weixin_47482194/article/details/106528032)