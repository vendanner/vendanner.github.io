---
layout:     post
title:      SparkSQL 访问 Hive
subtitle:   SQL
date:       2019-10-11
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - SparkSQL
    - Hive
    - Spark
---

> https://spark.apache.org/docs/latest/sql-data-sources-hive-tables.html

### `Metastore`

![](https://vendanner.github.io/img/SparkSQL/hive_metasore_spark.png)

`hive ` 启动 `bin/hive --service metastore` 。默认监听 `9083` 端口

```shell
[hadoop@danner000 ~]$ ps -ef| grep hive
hadoop   123666  36123 11 09:29 pts/2    00:00:46 /home/hadoop/app/jdk1.8.0_45/bin/java -Xmx256m -Djava.net.preferIPv4Stack=true -Dhadoop.log.dir=/home/hadoop/app/hadoop-2.6.0-cdh5.15.1/logs -Dhadoop.log.file=hadoop.log -Dhadoop.home.dir=/home/hadoop/app/hadoop-2.6.0-cdh5.15.1 -Dhadoop.id.str=hadoop -Dhadoop.root.logger=INFO,console -Djava.library.path=/home/hadoop/app/hadoop-2.6.0-cdh5.15.1/lib/native -Dhadoop.policy.file=hadoop-policy.xml -Djava.net.preferIPv4Stack=true -Xmx512m -Dhadoop.security.logger=INFO,NullAppender org.apache.hadoop.util.RunJar /home/hadoop/app/hive/lib/hive-service-1.1.0-cdh5.15.1.jar org.apache.hadoop.hive.metastore.HiveMetaStore
hadoop   123885  35322  5 09:36 pts/0    00:00:00 grep hive
[hadoop@danner000 ~]$ netstat -nlp | grep 123666
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:9083                0.0.0.0:*                   LISTEN      123666/java  
```

### 配置环境

`hadoop` 的 `core-site.xml`，`hive` 的 `hive-site.xml` 拷贝到 `project` 下的 `Resource`。

### 添加依赖

```xml
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-hive_2.11</artifactId>
      <version>${spark.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-core_2.11</artifactId>
      <version>${spark.version}</version>
    </dependency>
```

### `Code`

```scala
object ForamtApp {

    /**
      * hive :
      *     将 HDFS 下 core-site.xml 和 hive下 hive-site.xml 复制到 res
      *		配置 spark-hive_xxx 依赖
      * @param spark
      */
    def hive(spark: SparkSession):Unit = {
        import spark.implicits._

        spark.sql("use danner_db")
        spark.sql("show tables").show()
        spark.table("platform_stat").show()
    }
    def main(args: Array[String]): Unit = {
	
        // 配置访问用户
        System.setProperty("HADOOP_USER_NAME","hadoop")
        
        // Hive 操作要设置 enableHiveSupport
        val spark = SparkSession.builder().appName("ForamtApp").master("local[2]")
                .config("hive.metastore.uris", "thrift://192.168.22.147:9083")
                .enableHiveSupport().getOrCreate()

        hive(spark)

        spark.close()
    }
}
```

- 配置 `Hadoop` 用户

- `enableHiveSupport`

- `hive.metastore.uris` 可以在代码设置也可以在 `hive-site.xml`设置，若是默认端口 `9083`可以不设置

  