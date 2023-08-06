---
layout:     post
title:      Mac duckdb 测试环境
subtitle:
date:       2023-05-26
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - duckdb
    - db
---

老规矩：Mac 上开发，docker 上编译测试；这次直接在 Mac 上安装启动docker。

### git code

在github 上fork [duckdb 代码](https://github.com/duckdb/duckdb)后，在mac 上git code

```shell
git clone https://github.com/user/duckdb.git

git remote add upstream https://github.com/duckdb/duckdb.git    # 后续代码同步 pull
git fetch upstream master

git checkout -b RTFSC   # 拉个分支出来测试
```

### docker

在Mac 安装docker [doesktop](https://www.docker.com/products/docker-desktop/) 。注意m1/m2 要选择Apple Chip

挂载duckdb 代码，启动一个 `centos 7` 镜像的docker contianer

```shell
docker pull centos:7
docker run -it \
-v /用户目录/duckdb:/root/duckdb # git clone 代码挂载到容器 /root/duckdb 目录
--name duckdb-test 
-d centos:latest

# 进入docker
docker exec -it duckdb-test /bin/bash
```

### 编译

```shell
# 安装一些依赖
 yum install cmake -y
 yum install make -y
 yum install -y python3
 yum install gcc gcc-c++ -y
 
 # build 
make debug
```

### 测试

编译成功会生成 `build` 目录，进入 `build/debug` ，执行

```shell
[root@4aa492bb9986 debug]# ./duckdb
v0.0.1-dev0
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D
```

查询`csv` 文件

```shell
D SELECT * FROM read_csv_auto('/root/duckdb/data/csv/dirty_line.csv');
| COL1 | COL2 | COL3 |
|------|------|------|
| 1.5  | a    | 3    |
| 2.5  | b    | 4    |
D SELECT sum(COL3) FROM read_csv_auto('/root/duckdb/data/csv/dirty_line.csv');
| sum(COL3) |
|-----------|
| 7         |
D
```

### 代码修改

Mac 上 Clion 直接打开duckdb 根目录，等待自动加载；加载完成后代码点击后能自动跳转，可以开始阅读和修改代码。

修改后，在docker 上编译和测试。

