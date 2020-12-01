---
layout:     post
title:      Flink SQL 之 Create Table
subtitle:   
date:       2020-08-10
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - SQL
    - Table
---

基于 1.10

在 Flink SQL 中，我们可以很简单的定义 `Source` 和 `Sink`。如下所示

```sql
CREATE TABLE user_log (
    user_id VARCHAR,
    item_id VARCHAR,
    category_id VARCHAR,
    behavior VARCHAR,
    ts TIMESTAMP
) WITH (
    'connector.type' = 'kafka', -- 使用 kafka connector
    'connector.version' = 'universal',  -- kafka 版本，universal 支持 0.11 以上的版本
    'connector.topic' = 'user_behavior',  -- kafka topic
    'connector.startup-mode' = 'earliest-offset', -- 从起始 offset 开始读取
    'connector.properties.0.key' = 'zookeeper.connect',  -- 连接信息
    'connector.properties.0.value' = 'localhost:2181', 
    'connector.properties.1.key' = 'bootstrap.servers',
    'connector.properties.1.value' = 'localhost:9092', 
    'update-mode' = 'append',
    'format.type' = 'json',  -- 数据源格式为 json
    'format.derive-schema' = 'true' -- 从 DDL schema 确定 json 解析规则
)

CREATE TABLE pvuv_sink (
    dt VARCHAR,
    pv BIGINT,
    uv BIGINT
) WITH (
    'connector.type' = 'jdbc', -- 使用 jdbc connector
    'connector.url' = 'jdbc:mysql://localhost:3306/flink-test', -- jdbc url
    'connector.table' = 'pvuv_sink', -- 表名
    'connector.username' = 'root', -- 用户名
    'connector.password' = '123456', -- 密码
    'connector.write.flush.max-rows' = '1' -- 默认5000条
)
```

### SPI

在使用方便的同时，我们有必要深入理解下底层原理。Flink 会通过 **SPI 机制**将 classpath 下注册的所有工厂类加载进来。本文中主要涉及 `Source` 和 `Sink` 类，你可以在 `flink-connectors` 这个子模块找到原生支持的 `TableSourceSinkFactory`。

#### 加载

sql 创建一个 table ，底层会通过 `TableFactoryService` 根据创建时的属性寻找匹配 SourceSink 。

```java
// org.apache.flink.table.factories.TableFactoryUtil
public static <T> TableSink<T> findAndCreateTableSink(
    @Nullable Catalog catalog,
    ObjectIdentifier objectIdentifier,
    CatalogTable catalogTable,
    ReadableConfig configuration,
    boolean isStreamingMode) {
  TableSinkFactory.Context context = new TableSinkFactoryContextImpl(
    objectIdentifier,
    catalogTable,
    configuration,
    !isStreamingMode);
  if (catalog == null) {
    return findAndCreateTableSink(context);
  } else {
    return createTableSinkForCatalogTable(catalog, context)
      .orElseGet(() -> findAndCreateTableSink(context));
  }
}
public static <T> TableSink<T> findAndCreateTableSink(TableSinkFactory.Context context) {
  try {
    return TableFactoryService
        .find(TableSinkFactory.class, context.getTable().toProperties())
        .createTableSink(context);
  } catch (Throwable t) {
    throw new TableException("findAndCreateTableSink failed.", t);
  }
}
// org.apache.flink.table.factories.TableFactoryService
/**
  * Finds a table factory of the given class and property map.
  *
  * @param factoryClass desired factory class
  * @param propertyMap properties that describe the factory configuration
  * @param <T> factory class type
  * @return the matching factory
  */
public static <T extends TableFactory> T find(Class<T> factoryClass, Map<String, String> propertyMap) {
  return findSingleInternal(factoryClass, propertyMap, Optional.empty());
}
// 本例中 factoryClass = org.apache.flink.table.factories.TableFactory
private static <T extends TableFactory> T findSingleInternal(
    Class<T> factoryClass,
    Map<String, String> properties,
    Optional<ClassLoader> classLoader) {
  // 加载利用 SPI 方式注册的 TableFactory 类
  List<TableFactory> tableFactories = discoverFactories(classLoader);
  // 匹配符合 connector 的类
  List<T> filtered = filter(tableFactories, factoryClass, properties);
  // 最后晒选符合条件的 TableFactory 不能超过一个，否则报错
  if (filtered.size() > 1) {
    throw new AmbiguousTableFactoryException(
      filtered,
      factoryClass,
      tableFactories,
      properties);
  } else {
    return filtered.get(0);
  }
}
```



```java
// org.apache.flink.table.factories.TableFactoryService
/**
  * Searches for factories using Java service providers.
  */
private static List<TableFactory> discoverFactories(Optional<ClassLoader> classLoader) {
  try {
    List<TableFactory> result = new LinkedList<>();
    ClassLoader cl = classLoader.orElse(Thread.currentThread().getContextClassLoader());
    ServiceLoader
      .load(TableFactory.class, cl)
      .iterator()
      // 准备好加载器，遍历
      .forEachRemaining(result::add);
    return result;
  } catch (ServiceConfigurationError e) {
    LOG.error("Could not load service provider for table factories.", e);
    throw new TableException("Could not load service provider for table factories.", e);
  }
}
// java.util.ServiceLoader
/**
  * Creates a new service loader for the given service type and class
  * loader.
  *
  * @param  <S> the class of the service type
  *
  * @param  service
  *         The interface or abstract class representing the service
  *
  * @param  loader
  *         The class loader to be used to load provider-configuration files
  *         and provider classes, or <tt>null</tt> if the system class
  *         loader (or, failing that, the bootstrap class loader) is to be
  *         used
  *
  * @return A new service loader
  */
public static <S> ServiceLoader<S> load(Class<S> service,
                                        ClassLoader loader)
{
    return new ServiceLoader<>(service, loader);
}
private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}
public void reload() {
   // 每次都重新加载
    providers.clear();
    // service = org.apache.flink.table.factories.TableFactory
    lookupIterator = new LazyIterator(service, loader);
}
// 遍历
default void forEachRemaining(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    // 遍历
    while (hasNext())
        // 实例化
        action.accept(next());
}
// java.util.ServiceLoader.LazyIterator
public boolean hasNext() {
    if (acc == null) {
        return hasNextService();
    } else {
        PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
            public Boolean run() { return hasNextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
}
private boolean hasNextService() {
  if (nextName != null) {
      return true;
  }
  if (configs == null) {
      try {
          // PREFIX = META-INF/services/
          // service.getName() 为TableFactory的全路径名 = org.apache.flink.table.factories.TableFactory
          // META-INF/services/org.apache.flink.table.factories.TableFactor 文件
          String fullName = PREFIX + service.getName();
          // 使用 classloader 加载所有 jar 中 resource 的 fullName 文件
          if (loader == null)
              configs = ClassLoader.getSystemResources(fullName);
          else
              configs = loader.getResources(fullName);
      } catch (IOException x) {
          fail(service, "Error locating configuration files", x);
      }
  }
  while ((pending == null) || !pending.hasNext()) {
      if (!configs.hasMoreElements()) {
          return false;
      }
     // configs.nextElement() 遍历之前读取的所有配置文件，
     // 读取一个 jar 的fullName 文件内容，按行进行解析
      pending = parse(service, configs.nextElement());
  }
  nextName = pending.next();
  return true;
}
// 加载并实例化类
private S nextService() {
  if (!hasNextService())
      throw new NoSuchElementException();
  // nextName = META-INF/services/org.apache.flink.table.factories.TableFactor 文件中一行的内容
  String cn = nextName;
  nextName = null;
  Class<?> c = null;
  try {
      c = Class.forName(cn, false, loader);
  } catch (ClassNotFoundException x) {
      fail(service,
            "Provider " + cn + " not found");
  }
  if (!service.isAssignableFrom(c)) {
      fail(service,
            "Provider " + cn  + " not a subtype");
  }
  try {
      S p = service.cast(c.newInstance());
      providers.put(cn, p);
      return p;
  } catch (Throwable x) {
      fail(service,
            "Provider " + cn + " could not be instantiated",
            x);
  }
  throw new Error();          // This cannot happen
}
```

- "META-INF/services/" + 根据传入的 factoryClass 全路径名称 =   fullName
- 读取所有资源的 fullname，并按行解析成 className
- 实例化 class，最后返回 List<TableFactory>

#### 筛选

```java
// org.apache.flink.table.factories.TableFactoryService
/**
  * Filters found factories by factory class and with matching context.
  */
private static <T extends TableFactory> List<T> filter(
    List<TableFactory> foundFactories,
    Class<T> factoryClass,
    Map<String, String> properties) {
  ...
  // 从 foundFactories 过滤出是 factoryClass 的类
  List<T> classFactories = filterByFactoryClass(
    factoryClass,
    properties,
    foundFactories);
  // 获取类的 requiredContext 属性与 properties 匹配
  // 以 kafka 为例，requiredContext：connector.type = kafka
  // requiredContext 是 TableFactory 的抽象函数，所有 TableFactory 都会实现
  List<T> contextFactories = filterByContext(
    factoryClass,
    properties,
    classFactories);
  // 其他 connector 属性(supportedProperties)匹配
  return filterBySupportedProperties(
    factoryClass,
    properties,
    classFactories,
    contextFactories);
}
private static <T extends TableFactory> List<T> filterBySupportedProperties(
    Class<T> factoryClass,
    Map<String, String> properties,
    List<T> classFactories,
    List<T> contextFactories) {

  final List<String> plainGivenKeys = new LinkedList<>();
  properties.keySet().forEach(k -> {
    // replace arrays with wildcard
    String key = k.replaceAll(".\\d+", ".#");
    // ignore duplicates
    if (!plainGivenKeys.contains(key)) {
      plainGivenKeys.add(key);
    }
  });

  List<T> supportedFactories = new LinkedList<>();
  Tuple2<T, List<String>> bestMatched = null;
  for (T factory: contextFactories) {
    Set<String> requiredContextKeys = normalizeContext(factory).keySet();
    // 获取当前 factory 支持的属性
    Tuple2<List<String>, List<String>> tuple2 = normalizeSupportedProperties(factory);
    // 从 sql 创建时定义的属性中去除 requiredContex 属性，之前已判断
    List<String> givenContextFreeKeys = plainGivenKeys.stream()
      .filter(p -> !requiredContextKeys.contains(p))
      .collect(Collectors.toList());
    // TableFormatFactory 校验，本例不需要
    List<String> givenFilteredKeys = filterSupportedPropertiesFactorySpecific(
      factory,
      givenContextFreeKeys);
    // 校验 sql 中的属性都是改 factory 支持的
    boolean allTrue = true;
    List<String> unsupportedKeys = new ArrayList<>();
    for (String k : givenFilteredKeys) {
      if (!(tuple2.f0.contains(k) || tuple2.f1.stream().anyMatch(k::startsWith))) {
        allTrue = false;
        unsupportedKeys.add(k);
      }
    }
    // 都支持加入候选
    if (allTrue) {
      supportedFactories.add(factory);
    } else {
      if (bestMatched == null || unsupportedKeys.size() < bestMatched.f1.size()) {
        bestMatched = new Tuple2<>(factory, unsupportedKeys);
      }
    }
  }
  // 没找到合适的 TableFactory，输出报错信息
  ...
  return supportedFactories;
}

```

- 筛选 foundFactories 是 TableFactory
- 筛选 requiredContext 属性
- 判断属性是否都在 SupportedProperties

### 定制

#### 问题

flink sql 的 group by 结果写入到 kafka 时，报错

```shell
Exception in thread "main" org.apache.flink.table.api.TableException: AppendStreamTableSink requires that Table has only insert changes.
```

Kafka Sink 是 append 流，而 group by 是[回撤流](https://vendanner.github.io/2020/06/02/Flink-%E4%B9%8B-Retract/)无法写到 kafka。回撤流和追加流的区别在于多了个状态，我们只需要 状态 = True  的数据；核心代码如下

```java
// 将 kafka 改成 upsert
@Override
public DataStreamSink<?> consumeDataStream(DataStream<Tuple2<Boolean, Row>> dataStream) {
    final SinkFunction<Row> kafkaProducer = createKafkaProducer(
            topic,
            properties,
            serializationSchema,
            partitioner);
    return dataStream.filter(t -> t.f0).map(t -> t.f1)
            .addSink(kafkaProducer)
            .setParallelism(dataStream.getParallelism())
            .name(TableConnectorUtils.generateRuntimeName(this.getClass(), getFieldNames()));
}
```

#### 配置

尽量不改源码，我们新建 TableSink

- 将 KafkaValidator、KafkaTableSink、KafkaTableSinkBase、KafkaTableSourceSinkFactory、KafkaTableSourceSinkFactoryBase 代码复制并修改为自定义的类名
- KafkaTableSourceSinkFactoryBase `requiredContext` 函数将 `connector.type` 改成自定义的类型，后续使用时指定
- KafkaTableSinkBase 的 `consumeDataStream` 改成上面的函数，以支持回撤流
- 在工程的 Resource 目录下创建 META-INF/services/org.apache.flink.table.factories.TableFactor 文件，并写入刚刚自定义的 KafkaTableSourceSinkFactory；此处是注册





## 参考资料：

[【翻译】Flink Table API & SQL 自定义 Source & Sink](https://www.cnblogs.com/Springmoon-venn/p/12609011.html)

[Java SPI机制在Flink SQL中的应用](https://juejin.im/post/6854573215633801230)

