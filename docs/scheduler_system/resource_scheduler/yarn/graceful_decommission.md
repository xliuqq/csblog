# Yarn节点上下线

> 官方文档：https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/GracefulDecommission.html

## 概述

YARN 很容易扩展：任何新的 NodeManager 都可以加入配置的 ResourceManager 并开始执行作业。。但为了实现完全弹性，我们需要一个停用过程，这有助于删除现有节点并缩小集群规模。

节点停用有两种策略：

- `Normal Decommission` ：立即关闭
- `GRACEFUL Decommission `：不会在其上调度新容器并等待正在运行的容器完成（或超时），再将节点转换为 DECOMMISSIONED



## Normal Decommission

步骤：（不需要重启 ResourceManager）

- 将属性`yarn.resourcemanager.nodes.exclude-path` 添加到 `yarn-site.xml`

- 创建一个文本文件（位置在上一步中定义），其中一行包含所选 NodeManager 的名称
- 执行命令：`./bin/yarn rmadmin -refreshNode`



## GRACEFUL Decommission

> 默认 client 端操作，阻塞。

同 `Normal Decommission`， 只是需要配置 `-g timeout` 参数。
