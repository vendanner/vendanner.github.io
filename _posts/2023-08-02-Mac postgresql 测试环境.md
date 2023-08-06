---
layout:     post
title:      Mac postgresql 测试环境
subtitle:
date:       2023-08-02
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - pg
    - db
---

老规矩：Mac 上开发，docker 上编译测试；这次直接在 Mac 上安装启动docker。

### git code

在github 上fork [pg 代码](https://github.com/postgres/postgres)后，在mac 上git code

```shell
git clone https://github.com/user/postgres.git

# fork 后丢失其他分支，为了能拉取 REL_15_STABLE 分支
git remote add upstream https://github.com/postgres/postgres.git    
git fetch upstream master

git fetch upstream
git checkout -b REL_15_STABLE upstream/REL_15_STABLE
git checkout -b 15_RTFSC		# 拉15分支出来测试

```

### docker

启动一个 `centos 7` 镜像的docker contianer

```shell
docker pull centos:7
docker run -it \
-p 15432:5432 \
-v /用户目录/postgres:/home/postgres/postgresql-15 \  # git clone 代码挂载到容器 /root/home/postgres 目录
--name pg-test \
-d centos:centos7

# 进入docker
docker exec -it pg-test /bin/bash
```

### 编译

#### 用户

```shell
adduser postgres  
passwd postgres
chown -R postgres:root /home/postgres
PATH=/home/postgres/postgresql-12.1/prebuild/bin:$PATH:$HOME/.local/bin:$HOME/bin
```

#### 安装依赖

```shell
 yum install cmake -y
 yum install make -y
 yum install gcc gcc-c++ -y
 
yum install -y bison \
flex \
readline-devel \
zlib-devel \
openssl-devel \
libxml2-devel \
libxslt-devel \
uuid-devel \
openldap-devel \
python-devel \
krb5-devel \
tcl-devel \
pam-devel \
gettext-devel \
gcc-c++ \
gtk2-devel \
automake \
gettext\
perl-ExtUtils-Embed 
```

#### 配置

```shell
cd  /home/postgres/postgresql-15/
./configure --enable-debug --enable-cassert --prefix=/home/postgres/postgresql-15/prebuild CFLAGS=-O0

mkdir prebuild

vi ~/.bash_profile
PATH=/home/postgres/postgresql-15/prebuild/bin:$PATH:$HOME/.local/bin:$HOME/bin
```

####  编译

```shell
cd  /home/postgres/postgresql-15/
make -j8
make install
```

### 测试

#### 建库

```shell
# 初始化postgres 集群
mkdir -p /home/postgres/data15
initdb -k -D /home/postgres/data15
# 配置文件 postgresql.conf 在data15 目录里

# 在docker 发现 initdb: error: invalid locale settings; check LANG and LC_* environment variables
vi /etc/bashrc
export LC_ALL="en_US.UTF-8"
# 退出 /etc/bashrc

localedef -i en_US -f UTF-8 en_US.UTF-8
```

#### 启动

```shell
pg_ctl -D /home/postgres/data15 -l logfile start		# 启动
pg_ctl -D /home/postgres/data15 -l logfile stop		  # 关闭
```

#### 连接

```shell
psql -p5432		# 端口默认 5432

# 测试 sql
select * from pg_settings where name='port';
```



