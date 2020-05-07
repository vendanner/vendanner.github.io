---
layout:     post
title:      访问带 kerberos 认证的 Hive
subtitle:   HDFS
date:       2020-04-24
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 大数据
    - kerberos
    - Hive
---

集群开启 `kerberos` ，下午一顿折腾才能远程访问 Hive，特此记录。mac 环境

### keytab

kerberos 访问是要验证 `keytab`，那么就必须要在本地生成对应的文件

- 拷贝服务器上的 `krb5.conf`，下载到本地 `/etc/krb5.conf`
- 拷贝服务器上的 `xxx.keytab`，下载到本地并执行命令获取和缓存 principal（当前主体）初始的票据授予票据（TGT）

```shell
kinit -kt xxx.keytab you_principal
klist			
# 显示当前机器的 principal
Ticket cache: KCM:0
Default principal: you_principal

Valid starting       Expires              Service principal
04/24/2020 19:20:03  04/25/2020 19:20:03  krbtgt/xxx
	renew until 05/09/2020 19:20:03
```

### xml

在当前工程的资源文件夹下配置

- Core-site.xml

```xml
...
<property>
  <name>hadoop.http.authentication.kerberos.principal</name>
  <value>you_principal</value>
</property>
<property>
  <name>hadoop.http.authentication.kerberos.keytab</name>
  <value>/.../xxx.keytab</value>
</property>
...
```

- Hive-site.xml

```xml
...
<!-- default_realm 是 krb5.conf 中key 的值-->
<property>
  <name>hive.metastore.kerberos.principal</name>
  <value>hive/_HOST@default_realm</value>
</property>
<property>
  <name>hive.server2.authentication.kerberos.principal</name>
  <value>hive/_HOST@default_realm</value>
</property>
...
<!-- 注意涉及到的服务器都要用 hostname，然后在 /etc/hosts 配置；千万别用 IP-->
```

### code

```java
 System.setProperty("java.security.krb5.conf","/etc/krb5.conf");
```

以上全部配置结束后，就可以写代码了，没有什么特殊的与不带 kerberos 一样。



## 参考资料

[Kerberos之Beeline客户端参数解析](https://www.jianshu.com/p/79626006351d)

[Kerberos 错误消息](https://docs.oracle.com/cd/E26926_01/html/E25889/trouble-2.html)