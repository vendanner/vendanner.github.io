---
layout:     post
title:      Spark on YARN 加速启动
subtitle:   
date:       2019-10-18
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spark
    - YARN
    - bigdata
---

### 背景

`Spark on YARN` 每次启动时会将本地的 `spark` `jar` 和 `conf `上传到 `HDFS`，这样会消耗很长的时间

```shell
[hadoop@danner000 jars]$  spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster spark-examples_2.11-2.4.4.jar 3
...
`19/10/18 13:58:26 WARN yarn.Client: Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME.`
`19/10/18 13:58:30 INFO yarn.Client: Uploading resource file:/tmp/spark-294ab9b7-97ff-4ffa-8e4f-ae44a89dd5da/__spark_libs__1410305138065236635.zip -> hdfs://192.168.22.147:9000/user/hadoop/.sparkStaging/application_1571146456067_0024/__spark_libs__1410305138065236635.zip`
19/10/18 13:58:44 INFO yarn.Client: Uploading resource file:/home/hadoop/app/spark-2.4.4-bin-2.6.0-cdh5.15.1/examples/jars/spark-examples_2.11-2.4.4.jar -> hdfs://192.168.22.147:9000/user/hadoop/.sparkStaging/application_1571146456067_0024/spark-examples_2.11-2.4.4.jar
19/10/18 13:58:45 INFO yarn.Client: Uploading resource file:/tmp/spark-294ab9b7-97ff-4ffa-8e4f-ae44a89dd5da/__spark_conf__5888474803491307773.zip -> hdfs://192.168.22.147:9000/user/hadoop/.sparkStaging/application_1571146456067_0024/__spark_conf__.zip
...
```

查看上面日志，是由于没有设置 `spark.yarn.archive` 或 `spark.yarn.jars`，所以每次启动的时候都会上传`libs`

### 优化

既然知道是哪个属性的原因，那我们就从源码里看看如何设置

```scala
// org.apache.spark.deploy.yarn.Client
private def createContainerLaunchContext {
    ... 
    val appStagingDirPath = new Path(appStagingBaseDir, getAppStagingDir(appId))
    ...
    val localResources = prepareLocalResources(appStagingDirPath, pySparkArchives)
    ...
}
// org.apache.spark.deploy.yarn.Client
private[yarn] def copyFileToRemote {
    val destFs = destDir.getFileSystem(hadoopConf)
    val srcFs = srcPath.getFileSystem(hadoopConf)
    var destPath = srcPath
    if (force || !compareFs(srcFs, destFs) || "file".equals(srcFs.getScheme)) {
      destPath = new Path(destDir, destName.getOrElse(srcPath.getName()))
      logInfo(s"Uploading resource $srcPath -> $destPath")
      FileUtil.copy(srcFs, srcPath, destFs, destPath, false, hadoopConf)
      destFs.setReplication(destPath, replication)
      destFs.setPermission(destPath, new FsPermission(APP_FILE_PERMISSION))
    } else {
      logInfo(s"Source and destination file systems are the same. Not copying $srcPath")
    }
    ...
}
def prepareLocalResources {
    ...
    
    def distribute {
      val trimmedPath = path.trim()
      val localURI = Utils.resolveURI(trimmedPath)
      if (localURI.getScheme != LOCAL_SCHEME) {
        if (addDistributedUri(localURI)) {
          val localPath = getQualifiedLocalPath(localURI, hadoopConf)
          val linkname = targetDir.map(_ + "/").getOrElse("") +
           destName.orElse(Option(localURI.getFragment())).getOrElse(localPath.getName())
          val destPath = copyFileToRemote(destDir, localPath, replication, symlinkCache)
          val destFs = FileSystem.get(destPath.toUri(), hadoopConf)
          distCacheMgr.addResource(
            destFs, hadoopConf, destPath, localResources, resType, linkname, statCache,
            appMasterOnly = appMasterOnly)
          (false, linkname)
        } else {
          (false, null)
        }
      } else {
        (true, trimmedPath)
      }
    }
    ...
    /**
     * Add Spark to the cache. There are two settings that control what files to add to the cache:
     * - if a Spark archive is defined, use the archive. The archive is expected to contain
     *   jar files at its root directory.
     * - if a list of jars is provided, filter the non-local ones, resolve globs, and
     *   add the found files to the cache.
     *
     * Note that the archive cannot be a "local" URI. If none of the above settings are found,
     * then upload all files found in $SPARK_HOME/jars.
     */
    val sparkArchive = sparkConf.get(SPARK_ARCHIVE)
    if (sparkArchive.isDefined) {
      val archive = sparkArchive.get
      require(!isLocalUri(archive), s"${SPARK_ARCHIVE.key} cannot be a local URI.")
      distribute(Utils.resolveURI(archive).toString,
        resType = LocalResourceType.ARCHIVE,
        destName = Some(LOCALIZED_LIB_DIR))
    }else {
      sparkConf.get(SPARK_JARS) match {
      	case Some(jars) =>{
            // 操作类似 SPARK_ARCHIVE，把 SPARK_JARS 上传；两者设置一个即可
            ... 
        }
       case None =>
          // No configuration, so fall back to uploading local jar files.
          logWarning(s"Neither ${SPARK_JARS.key} nor ${SPARK_ARCHIVE.key} is set, falling back " + "to uploading libraries under SPARK_HOME.")
          // 把 spark/jars 所有 jar 上传
      }
       
    ...
}
```

- 设置 `SPARK_ARCHIVE` 后，将 `SPARK_ARCHIVE`  目录分发 (distribute)
- `distribute` 中判断是否为本地文件和是否已上传
- `copyFileToRemote` 判断原文件和目标文件是否为**同个文件系统**，若相同则不上传
- `destPath` 在 `createContainerLaunchContext` 函数被赋值 `appStagingDirPath`，根据 [hadoop job 执行流程](https://vendanner.github.io/2019/08/25/hadoop-job-%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B/) 可知 `StagingDir` 是 `Yarn job` 为执行任务存放文件而临时创建的目录；在本案例中就是 `HDFS` 目录

由以上分析可知，只需将 `SPARK_ARCHIVE`  设置为 `hdfs` 目录就可以避免每次上传的困扰。

- 将 `spark/jars/*.jar` 打包成 `zip` 并上传到 `HDFS`

```shell
spark-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster `--conf spark.yarn.archive=hdfs://192.168.22.147:9000/lib/dep/spark/spark_jar.zip` spark-examples_2.11-2.4.4.jar 3
...
19/10/18 12:39:16 INFO yarn.Client: Preparing resources for our AM container
19/10/18 12:39:16 INFO yarn.Client: `Source and destination file systems are the same. Not copying hdfs://192.168.22.147:9000/lib/dep/spark/spark_jar.tar`
...
```

> `spark.yarn.archive` 也可以设置在 `spark-defaults.conf` ; `spark.yarn.jars` 相同操作，两者等效

### `Conf`

看日志可知，每次启动也都上传，它会上传 `SPARK_CONF_DIR` 和 `HADOOP_CONF_DIR`目录下的文件。但此 `Conf` 无法优化，因为就是算是源文件和目标文件在同个文件系统，也会**强制复制**

```scala
    // This code forces the archive to be copied, so that unit tests pass (since in that case both
    // file systems are the same and the archive wouldn't normally be copied). In most (all?)
    // deployments, the archive would be copied anyway, since it's a temp file in the local file
    // system.
    val remoteConfArchivePath = new Path(destDir, LOCALIZED_CONF_ARCHIVE)
    val remoteFs = FileSystem.get(remoteConfArchivePath.toUri(), hadoopConf)
    sparkConf.set(CACHED_CONF_ARCHIVE, remoteConfArchivePath.toString())

    val localConfArchive = new Path(createConfArchive().toURI())
    copyFileToRemote(destDir, localConfArchive, replication, symlinkCache, force = true,
      destName = Some(LOCALIZED_CONF_ARCHIVE))
```

