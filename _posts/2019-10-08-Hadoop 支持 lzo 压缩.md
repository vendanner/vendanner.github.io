---
layout:     post
title:      Hadoop 支持 lzo 压缩
subtitle:   
date:       2019-10-08
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - HDFS
    - hadoop
    - LZO
    - bigdata
---

`Hadoop` 中最常用的压缩方式 `LZO`，因为它支持**分割**，**压缩/解压速度也比较快**，合理的压缩率 (关于 hadoop 压缩算法，参考[压缩算法]([https://vendanner.github.io/2019/08/21/G7-%E7%9F%A5%E8%AF%86-%E4%BA%8C/#%E5%8E%8B%E7%BC%A9%E7%AE%97%E6%B3%95](https://vendanner.github.io/2019/08/21/G7-知识-二/#压缩算法)))。 但 `Hadoop` 本身不支持 `LZO`，需要另行安装。

### 安装依赖

```shell
[root@danner000 hadoop]# yum install -y svn ncurses-devel
[root@danner000 hadoop]# yum install -y gcc gcc-c++ make cmake
[root@danner000 hadoop]# yum install -y openssl openssl-devel svn ncurses-devel zlib-devel libtool
[root@danner000 hadoop]# yum install -y lzo lzo-devel lzop autoconf automake cmake
```

### 编译 `LZO`

```shell
[hadoop@danner000 software]$ wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.06.tar.gz
[hadoop@danner000 software]$ tar -zxvf lzo-2.06.tar.gz -C ../app/
[hadoop@danner000 software]$ cd ../app/lzo-2.06/
[hadoop@danner000 lzo-2.06]$ export CFLAGS=-m64
[hadoop@danner000 lzo-2.06]$ mkdir compile
[hadoop@danner000 lzo-2.06]$ ./configure -enable-shared -prefix=/home/hadoop/app/lzo-2.06/compile
[hadoop@danner000 lzo-2.06]$ make &&  make install
```

### 编译 `Hadoop-lzo`

```shell
[hadoop@danner000 software]$ wget https://github.com/twitter/hadoop-lzo/archive/master.zip
[hadoop@danner000 software]$ unzip -d ../app/ master.zip
[hadoop@danner000 software]$ cd ../app/hadoop-lzo-master
```

```shell
# 修改 hadoop-lzo-master 目录下 pom.xml 文件

# 增加 repo
     <repository>
       <id>cloudera</id>
       <url>http://repository.cloudera.com/artifactory/cloudera-repos</url>
    </repository>
    
# 修改 hadoop 版本(匹配正在使用的 hadoop 版本)
<hadoop.current.version>2.6.0-cdh5.15.1</hadoop.current.version>
```

```shell
[hadoop@danner000 hadoop-lzo-master]$ export CFLAGS=-m64
[hadoop@danner000 hadoop-lzo-master]$ export CXXFLAGS=-m64
# 之前编译好的 lzo 文件
[hadoop@danner000 hadoop-lzo-master]$ export C_INCLUDE_PATH=/home/hadoop/app/lzo-2.06/compile/include
[hadoop@danner000 hadoop-lzo-master]$ export LIBRARY_PATH=/home/hadoop/app/lzo-2.06/compile/lib
# 编译
[hadoop@danner000 hadoop-lzo-master]$ mvn clean package -DskipTests
```

```shell
# 编译成功后
[hadoop@danner000 hadoop-lzo-master]$ cd target/native/Linux-amd64-64/
[hadoop@danner000 Linux-amd64-64]$ mkdir ~/app/hadoop-lzo-master/lzo-files
[hadoop@danner000 Linux-amd64-64]$ tar -cBf - -C lib . | tar -xBvf - -C ~/app/hadoop-lzo-master/lzo-files
./
./libgplcompression.la
./libgplcompression.so.0
./libgplcompression.so.0.0.0
./libgplcompression.a
./libgplcompression.so
# 复制到 hadoop/lib/native
[hadoop@danner000 Linux-amd64-64]$ cp ~/app/hadoop-lzo-master/lzo-files/libgplcompression* $HADOOP_HOME/lib/native/
# 复制 jar
[hadoop@danner000 hadoop-lzo-master]$ cp target/hadoop-lzo-0.4.21-SNAPSHOT.jar $HADOOP_HOME/share/hadoop/common
[hadoop@danner000 hadoop-lzo-master]$ cp target/hadoop-lzo-0.4.21-SNAPSHOT.jar $HADOOP_HOME/share/hadoop/mapreduce/lib
```

### 配置 `Hadoop`

```shell
# hadoop-env.sh 加下面配置
export LD_LIBRARY_PATH=/home/hadoop/app/lzo-2.06/compile/lib
```

```shell
# core-site.xml 加下面配置
<property>
    <name>io.compression.codecs</name>
    <value>org.apache.hadoop.io.compress.GzipCodec,
           org.apache.hadoop.io.compress.DefaultCodec,
           `com.hadoop.compression.lzo.LzoCodec,`
           `com.hadoop.compression.lzo.LzopCodec,`
           org.apache.hadoop.io.compress.BZip2Codec
        </value>
</property>
<property>
    <name>io.compression.codec.lzo.class</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
```

```xml
<!-- mapreduce-site.xml 加下面配置 --> 
    <property>
        <name>mapred.compress.map.output</name>
        <value>true</value>
    </property>

    <property>
        <name>mapred.map.output.compression.codec</name>
        <value>com.hadoop.compression.lzo.LzoCodec</value>
    </property>

    <property>
        <name>mapred.child.env</name>
        <value>LD_LIBRARY_PATH=/home/hadoop/app/lzo-2.06/compile/lib</value>
```

### 测试

准备文件

```shell
[hadoop@danner000 local]$ ls -lh ~/tmp/data/ | grep access
-rw-r--r--. 1 hadoop hadoop 859M Oct 15 21:49 access.txt
# lzo 压缩文件
[hadoop@danner000 local]$ lzop -f ~/tmp/data/access.txt
[hadoop@danner000 local]$ ls -lh ~/tmp/data/ | grep access
-rw-r--r--. 1 hadoop hadoop 859M Oct 15 21:49 access.txt
-rw-r--r--. 1 hadoop hadoop 750M Oct 15 21:49 access.txt.lzo
```

```shell
# 直接上传文件到 hdfs
[hadoop@danner000 local]$ hadoop fs -ls /input | grep access   
-rw-r--r--   1 hadoop supergroup  785566135 2019-10-15 21:54 /input/access.txt.lzo
# 执行 wc
[hadoop@danner000 local]$ hadoop jar /home/hadoop/app/hadoop-2.6.0-cdh5.15.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0-cdh5.15.1.jar wordcount -Dmapreduce.job.inputformat.class=com.hadoop.mapreduce.LzoTextInputFormat /input/access.txt.lzo /ou
19/10/15 22:02:27 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
19/10/15 22:02:30 INFO input.FileInputFormat: Total input paths to process : 1
`19/10/15 22:02:31 INFO mapreduce.JobSubmitter: number of splits:1`
19/10/15 22:02:31 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1571146456067_0001
19/10/15 22:02:32 INFO impl.YarnClientImpl: Submitted application application_1571146456067_0001
19/10/15 22:02:32 INFO mapreduce.Job: The url to track the job: http://danner000:8088/proxy/application_1571146456067_0001/
19/10/15 22:02:32 INFO mapreduce.Job: Running job: job_1571146456067_0001
# 观察到上面 splits=1，不支持分片
```

给 `lzo` 文件添加 `index` 后执行

```shell
# 用 hadoop-lzo.jar 给 lzo 文件生成 index
[hadoop@danner000 local]$hadoop jar /home/hadoop/app/hadoop-lzo-master/target/hadoop-lzo-0.4.21-SNAPSHOT.jar com.hadoop.compression.lzo.DistributedLzoIndexer /input/access.txt.lzo
[hadoop@danner000 local]$ hadoop fs -ls /input | grep access
-rw-r--r--   1 hadoop supergroup  785566135 2019-10-15 21:54 /input/access.txt.lzo
`-rw-r--r--   1 hadoop supergroup      27472 2019-10-15 22:06 /input/access.txt.lzo.index`
[hadoop@danner000 local]$ hadoop jar /home/hadoop/app/hadoop-2.6.0-cdh5.15.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0-cdh5.15.1.jar wordcount -Dmapreduce.job.inputformat.class=com.hadoop.mapreduce.LzoTextInputFormat /input/access.txt.lzo /ou
19/10/15 22:09:58 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
19/10/15 22:10:00 INFO input.FileInputFormat: Total input paths to process : 1
`19/10/15 22:10:00 INFO mapreduce.JobSubmitter: number of splits:6`
19/10/15 22:10:01 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1571146456067_0003
# ok，splits=6，也就是 map 有 6 个task
```









## 参照资料

[Hadoop支持lzo压缩（版本cdh5.15.1）](https://guguoyu.blog.csdn.net/article/details/102520589)