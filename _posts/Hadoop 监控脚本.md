---
layout:     post
title:      Hadoop 集群脚本
date:       2019-08-22
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - big data
    - Hadoop
    - shell
    - 监控
---

在编写集群脚本之前，先回顾下 `shell` 知识点

## Shell

### 入门

创建 `wc.sh`，并在文件写入以下内容：

> \#!/bin/bash

> echo "hello shell"

解释下上面两行代码：

- 第一行**注释**表明执行此 `shell` 是 `/bin/bash`；
- 第二行是 `shell` 具体要执行的脚本，输出 `hello shell`。

保存好文件内容后，增加**执行权限** `chmod u+x wc.sh` 后执行 `./wc.sh`,输出如下：
> hello shell

当然你也可以在首行加 `-x` 表示 `debug` 模式，会输出调试信息，整个文件内容:

> \#!/bin/bash -x
> 
> echo "hello shell"

执行后输出

> \+ echo 'hello shell' 
> hello shell

增加了调试信息 `+ echo 'hello shell' `，代表执行 `echo 'hello shell' ` 语句


### 变量定义与引用

创建 `variable.sh`，输入以下内容：

> \#!/bin/bash 
>
> hello='hello'
> date1='date1'
> date2=\`date\`
> word=word
> 
>echo $hello
>echo ${date1}
>echo ${date2}
>echo $word

执行输出：

> hello
> date1
> Thu Aug 22 23:10:34 CST 2019

关于变量，有几个知识点：

- `$` 后跟随的是**变量名**
- \`fun\` 是表示 `fun` 的返回值
- 字符串可以用 `""`、`''`包裹，也可以不用；但标准用法是用**引号**

当然也有需要**注意点**：

- `=` 前后不能有**空格**
- **变量名**最好用 `{}`，以免引起歧义
- **变量名**最好用**大写**