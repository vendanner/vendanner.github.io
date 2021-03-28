---
layout:     post
title:      Flink CheckPoint 之元数据管理
subtitle:   
date:       2021-02-13
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - CheckPoint
    - 源码
---







## 参考资料

[Flink 源码：Checkpoint 元数据详解](https://mp.weixin.qq.com/s/pVcLAPlfWT7P6cktOpGyIQ)

[Flink 清理过期 Checkpoint 目录的正确姿势](https://mp.weixin.qq.com/s/pQ2CSkKpQkaYQ2EcDSQBkg)

[Flink 源码：JM 端从 Checkpoint 恢复流程](https://mp.weixin.qq.com/s/eZ2FB5IBwrXMZyCoQ8elgw)

[Flink 源码：TM 端恢复及创建 OperatorState 的流程](https://mp.weixin.qq.com/s/6AcBWaQTLIoIMd1R1H_l7Q)

[Flink 源码：TM 端恢复及创建 KeyedState 的流程](https://mp.weixin.qq.com/s/DoYHxEFiCzvU3I7emWEKVg)