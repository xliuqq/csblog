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
- 系统*不允许出现跨名字空间的属主引用*。

示例：一个Deployment创建的 Pod

```yaml
apiVersion: v1
    ...
    ownerReferences:
    - apiVersion: apps/v1
      blockOwnerDeletion: true
      controller: true
	  # Deployment 创建 ReplicaSet，ReplicaSet创建Pod
      kind: ReplicaSet
      name: nginx-deployment-6b474476c4
      uid: 4fdcd81c-bd5d-41f7-97af-3a3b759af9a7
    ...
```



## 级联删除

当你删除某个对象A时，你可以控制 Kubernetes 是否去**自动删除依赖A的相关对象**， 这个过程称为**级联删除（Cascading Deletion）**。

### 前台级联删除

> kubectl 的 --cascade=foreground 参数指定。

正在被你删除的属主对象首先进入 **deletion in progress** 状态。 针对属主对象会发生以下事情：

- API Server 将对象的 `metadata.deletionTimestamp` 字段设置为该对象被标记为要删除的时间点；
- API Server 将对象的`metadata.finalizers` 字段设置为 `foregroundDeletion`；
- 在删除过程完成之前，通过 Kubernetes API **仍然可以看到该对象**。

当属主对象进入删除过程中状态后，控制器***删除依赖该属主的对象***。控制器在删除完所有依赖对象之后， 删除属主对象。

- 控制器必须在删除了拥有 `ownerReferences.blockOwnerDeletion=true` 的附属资源后，才能删除属主对象。

这时，通过 Kubernetes API 就无法再看到该对象。

### 后台级联删除（默认)

Kubernetes 服务器**立即删除属主对象**，控制器在后台清理所有依赖对象。 

- 依赖对象是拥有 `ownerReferences.blockOwnerDeletion=true` 的附属资源。

### 被遗弃的依赖对象（orphan）

> kubectl 的 --cascade=orphan 参数指定，*孤立* 这些依赖对象（即保留不删除）。

当 Kubernetes 删除某个属主对象时，被留下来的依赖对象被称作被遗弃的（Orphaned）对象。



## 未使用的容器和镜像

> 基于 [`KubeletConfiguration`](https://kubernetes.io/zh-cn/docs/reference/config-api/kubelet-config.v1beta1/) 进行配置。

kubelet 每 5 分钟对**未使用的镜像**执行一次垃圾收集， 每 1 分钟对**未使用的容器**执行一次垃圾收集。 

- 避免使用外部的垃圾收集工具，因为外部工具可能会破坏 kubelet 的行为，移除应该保留的容器。
- [周期时间](https://github.com/kubernetes/kubernetes/blob/8fe10dc378b7cc3b077b83aef86622e1019302d5/pkg/kubelet/kubelet.go#L1572)是在代码中固定的。

### 容器镜像生命周期

镜像管理器（Image Manager） 来管理所有镜像的生命周期， 该管理器是 kubelet 的一部分，工作时与 cadvisor 协同。

kubelet 在作出垃圾收集决定时会考虑如下时间和磁盘用量约束：

- `imageMinimumGCAge` ：默认'2m'，镜像的最小存在时间，避免在垃圾回收时删除过于新的镜像。
- `imageMaximumGCAge` ：默认"0s"（已禁用），表示镜像自上次使用后的最大存在时间（v1.30引入)。

- `imageGCHighThresholdPercent`：默认85，超出时出发垃圾收集，基于镜像上次被使用的时间来按顺序删除它们；
- `imageGCLowThresholdPercent`：默认80，持续删除镜像，直到磁盘用量达到该值为止；

### 容器垃圾收集

> kubelet 仅会回收由它所管理的容器。

kubelet 会基于如下变量对所有未使用的容器执行垃圾收集操作（启动参数，而不是配置文件）：

- `--minimum-container-ttl-duration`：垃圾回收某个容器时该容器的最少存活时间。设置为 `0` 表示没有限制（即容器可以被立即清理）。
- `--maximum-dead-containers-per-container`：每个 Pod 可以包含的已死亡的容器个数上限。默认值1，设置为小于 `0` 的值表示禁止使用此规则。
- `--maximum-dead-containers`：集群中可以存在的已死亡的容器个数上限。默认值：-1，禁止应用此规则。



## 节点压力驱逐

节点压力驱逐是 [kubelet](https://kubernetes.io/zh-cn/docs/reference/generated/kubelet) 主动终止 Pod 以回收节点上资源的过程。（v1.31 beta, default enabled)



### 弃用的 kubelet 垃圾收集功能

一些 kubelet 垃圾收集功能已被弃用，以鼓励使用驱逐机制。

| 现有标志                                  | 原因                                         |
| ----------------------------------------- | -------------------------------------------- |
| `--maximum-dead-containers`               | 一旦旧的日志存储在容器的上下文之外就会被弃用 |
| `--maximum-dead-containers-per-container` | 一旦旧的日志存储在容器的上下文之外就会被弃用 |
| `--minimum-container-ttl-duration`        | 一旦旧的日志存储在容器的上下文之外就会被弃用 |



## 参考文档

1. [垃圾收集 v.31 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/architecture/garbage-collection/).
2. [节点压力驱逐 v.31 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/)