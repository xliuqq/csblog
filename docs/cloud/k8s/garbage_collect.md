# 垃圾收集

垃圾收集允许系统清理如下资源：

- **终止的 Pod**（`phase`值为 `Succeeded` 或 `Failed`）
- **已完成的 Job**
- **不再存在属主引用的对象**
- **未使用的容器和容器镜像**
- 动态制备的、StorageClass 回收策略为 Delete 的 **PV 卷**
- 阻滞或者过期的 CertificateSigningRequest (CSR)
- 在以下情形中删除了的节点对象：
  - 当集群使用云控制器管理器运行于云端时；
  - 当集群使用类似于云控制器管理器的插件运行在本地环境中时。
- 节点租约对象



## 属主与依赖

属主引用（Owner Reference）可以告诉控制面哪些对象依赖于其他对象。

- 删除某对象时提供一个**清理关联资源**的机会。
- 系统不允许出现跨名字空间的属主引用。



## 级联删除

当你删除某个对象A时，你可以控制 Kubernetes 是否去**自动删除依赖A的相关对象**， 这个过程称为**级联删除（Cascading Deletion）**。

### 前台级联删除

> kubectl 的 --cascade=foreground 参数指定。

正在被你删除的属主对象首先进入 **deletion in progress** 状态。 针对属主对象会发生以下事情：

- API Server 将对象的 `metadata.deletionTimestamp` 字段设置为该对象被标记为要删除的时间点；
- API Server 将对象的`metadata.finalizers` 字段设置为 `foregroundDeletion`；
- 在删除过程完成之前，通过 Kubernetes API **仍然可以看到该对象**。

当属主对象进入删除过程中状态后，控制器删除其依赖对象。控制器在删除完所有依赖对象之后， 删除属主对象。

- 控制器必须在删除了拥有 `ownerReferences.blockOwnerDeletion=true` 的附属资源后，才能删除属主对象。

这时，通过 Kubernetes API 就无法再看到该对象。

### 后台级联删除（默认)

Kubernetes 服务器**立即删除属主对象**，控制器在后台清理所有依赖对象。 

- 依赖对象是拥有 `ownerReferences.blockOwnerDeletion=true` 的附属资源。

### 被遗弃的依赖对象（orphan）

> kubectl 的 --cascade=orphan 参数指定。

当 Kubernetes 删除某个属主对象时，被留下来的依赖对象被称作被遗弃的（Orphaned）对象。



## 未使用的容器和镜像

kubelet 每两分钟对**未使用的镜像**执行一次垃圾收集， 每分钟对**未使用的容器**执行一次垃圾收集。 

- 避免使用外部的垃圾收集工具，因为外部工具可能会破坏 kubelet 的行为，移除应该保留的容器。

### 容器镜像生命周期

镜像管理器（Image Manager） 来管理所有镜像的生命周期， 该管理器是 kubelet 的一部分，工作时与 cadvisor 协同。

kubelet 在作出垃圾收集决定时会考虑如下磁盘用量约束：

- `HighThresholdPercent`：超出时出发垃圾收集，基于镜像上次被使用的时间来按顺序删除它们；
- `LowThresholdPercent`：持续删除镜像，直到磁盘用量达到该值为止；

### 容器垃圾收集

> kubelet 仅会回收由它所管理的容器。

kubelet 会基于如下变量对所有未使用的容器执行垃圾收集操作：

- `MinAge`：垃圾回收某个容器时该容器的最小年龄。设置为 `0` 表示禁止使用此规则。
- `MaxPerPodContainer`：每个 Pod 可以包含的已死亡的容器个数上限。设置为小于 `0` 的值表示禁止使用此规则。
- `MaxContainers`：集群中可以存在的已死亡的容器个数上限。设置为小于 `0` 的值意味着禁止应用此规则。