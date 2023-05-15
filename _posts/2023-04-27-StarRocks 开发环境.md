---
layout:     post
title:      StarRocks 开发环境
subtitle:
date:       2023-04-27
author:     danner
header-img: img/bg.jpg
catalog: true
tags:
    - StarRocks
    - C++
---

主要是C/C++ 编译依赖于CPU，Java 没啥特别。

基本策略

- 开发/远程机代码同步是通过 git，IDE的 sync code 功能和git 分支切换，不太好使
- IDEA/Clion 连接远程机器，远程调试

### git

#### 开发机

按照 https://github.com/StarRocks/community/blob/main/Contributors/guide/workflow.md 文档，git code 到本地

```shell
git checkout -b danner_RTFSC
# 修改分支代码
touch danner 	# 新增文件
git add danner
git commit -m "add danner"
# 代码修改提交到 github 仓库
git push origin danner_RTFSC
```

#### 远程

git 同步代码，准备编译

```shell
git clone https://github.com/xxx/starrocks.git
git checkout -b danner_RTFSC origin/danner_RTFSC
# 后续有代码修改，直接
git pull
```

### Docker

准备编译环境，使用Docker - 官方已提供开发环境，但还需要设置参数。

细节看[参考资料一](https://www.inlighting.org/archives/setup-starrocks-development)

```dockerfile
FROM starrocks/dev-env-centos7:latest
# Your ssh password, default is xxx.
ARG password=123456
ARG starrockshome=/root/starrocks
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN yum -y install openssh-server && \
    ssh-keygen -A && \
    echo "root:${password}" |chpasswd && \
    sed -ri 's/^#?PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    yum -y install libffi-devel centos-release-scl vim openssl openssl-devel tmux && \
    yum -y install \ 
      devtoolset-11-gdb \
      rh-mysql57-mysql \
      devtoolset-11-gdb-gdbserver \
      rh-git227-git \
      debuginfo-install bzip2-libs-1.0.6-13.el7.x86_64 \
      elfutils-libelf-0.176-5.el7.x86_64 \
      elfutils-libs-0.176-5.el7.x86_64 \
      glibc-2.17-325.el7_9.x86_64 libattr-2.4.46-13.el7.x86_64 \
      libcap-2.22-11.el7.x86_64 systemd-libs-219-78.el7.x86_64 \
      xz-libs-5.2.2-1.el7.x86_64 zlib-1.2.7-20.el7_9.x86_64 && \
    # If your server is not in China, you can use GitHub.
    git clone https://github.com/rui314/mold.git /root/mold && \
    # git clone https://gitee.com/mirrors/mold.git /root/mold && \
    cd /root/mold && \
    git checkout v1.4.2 && \
    make -j$(nproc) && \
    make install && \
    ln -s /usr/local/bin/mold /usr/local/bin/ld
 
ENV STARROCKS_HOME ${starrockshome}
 
# Default ssh port is 3333, you can change it.
EXPOSE 3333
CMD ["/usr/sbin/sshd", "-D", "-p", "3333"]
```

根据Dockerfile build image

### 编译

#### Run

运行之前构建的 image

```shell
docker run -it -p 3333:3333 \
  -p 8131:8030 \
  -p 9121:9020 \
  -p 9131:9030 \
  -p 9111:9010 \
  -p 9161:9060 \
  -p 8140:8040 \
  -p 9151:9050 \
  -p 8160:8060 \
  -p 8001:8001 \
  -p 8003:8003 \
  --privileged \
  --cap-add SYS_PTRACE \
  -v ~/.m2:/root/.m2 \
  -v /宿主机/starrocks:/root/starrocks \
  --name sr-dev \
  -d starrocks/centos-danner:1.0
```

- /root/.m2：docker 中maven 本地仓库地址，持久化下载的Jar
- /root/starrocks： docker 中source code 地址
- -p：端口映射，FE/BE 远程调试使用；防止端口冲突

#### 进入

```shell
docker exec -it sr-dev /bin/bash
```

#### build

```shell
cd /root/starrocks/
./build.sh
```

第一次全量编译

### 调试

调试分为本地调试和远程调试

- 本地调试：代码执行在本地
  - FE：本地可执行
  - BE：x86 机器可执行，mac 上只能查看代码及代码跳转
- 远程调试：代码执行在远程机器
  - FE：IDEA Remote 功能
  - BE：gdb server + Clion remote gdb

#### IDEA

##### 本地 debug

缺少一些 `gensrc`，本地就懒的编译了，从远程机拷贝 - 搞个sftp 客户端，后续如果有修改要重新拷贝方便

- 将远程机 `fe/fe-core/target/generated-sources` 拷贝到本机 `fe/fe-core/target` 下
- 将远程机 `conf` 拷贝到本机 `fe/` 下
- 在本机 `fe/` 下新建 `meta` / `log` 文件夹

IDEA 打开本地 **fe目录**，等待自动加载完成。

配置运行环境变量(`Run/Debug Configurations`)

```shell
PID_DIR=/xxx/fe;STARROCKS_HOME=/xxx/fe;LOG_DIR=/xxx/fe/log
```

RUN `com.starrocks.StarRocksFE#main` ，输出如下表示本地环境已成功

```shell
notify new FE type transfer: LEADER
```

##### 远程debug

- 远程机器

StarRocks FE 下的启动脚本 `start_fe.sh` 自带**debug** 模式，但需要修改 **5005** 端口为 **8001**（docker 启动的时候端口已映射）

```shell
bin/start_fe.sh --debug
```

- 本机

IDEA 选择 `Remote JVM Debug`，填入`Host` 和 `Port` = 8001 ，会自动生成如下参数；点击**debug 按钮** 即可远程调试。

```shell
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8001
```

#### Clion

Clion 打开`starrocks` 目录，按**参考资料三** 设置build tools/cmake后，

在主目录新建 `CMakeLists.txt `，添加如下内容

```shell
cmake_minimum_required(VERSION 3.15)
project(StarRocks)

set(CMAKE_CXX_STANDARD 17)

include_directories(
        be/
        be/src
        be/test

        // thrift/protobuf files
        gensrc/build
        gensrc/build/common
        gensrc/build/gen_cpp

        // header files of third-parties
        ../deps/gperftools
        ../deps/include
)
include_directories(SYSTEM "/var/local/thirdparty/installed/include")  -- 远程机器上第三方目录，echo $STARROCKS_THIRDPARTY
include_directories(SYSTEM "/var/local/thirdparty/installed/gperftools/include")


add_executable(StarRocks
        be/src/service/starrocks_main.cpp)
```

build 即可，等待从远程机器下载**include 文件**，完成后可在Clion 进行代码跳转。

- 远程机器

修改 `start_backend.sh`，gdbserver 方式启动

```shell
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup ${START_BE_CMD} "$@" </dev/null &
else
   # exec ${START_BE_CMD} "$@" </dev/null ;修改之前
   gdbserver 0.0.0.0:8003 ${START_BE_CMD}
fi
```

执行 `bin/start_be.sh` 启动，可以在 be.out 看到如下信息

```shell
Process /root/starrocks/output/be/lib/starrocks_be created; pid = 25361
Listening on port 8003
```

- 本地机器

Clion 选择 `Remote debug`，填入'target remote args' 即可，例如 `xxx:8003`；保存后点击`debug 按钮` 开始调试。

### 修改

- 开发机修改代码，git push 提交到github
- 远程机git pull 拉取代码，编译运行





## 参考资料

[StarRocks Docker 开发环境搭建指南](https://www.inlighting.org/archives/setup-starrocks-development)

[第1.4章：FE开发环境搭建（拓展篇）](https://blog.csdn.net/ult_me/article/details/121582881)

[StarRocks 完美开发环境搭建](https://www.inlighting.org/archives/setup-perfect-starrocks-dev-env)

[code jump via Clion](https://github.com/StarRocks/community/blob/main/Contributors/guide/Clion.md#use-clion-to-load-starrocks-code)