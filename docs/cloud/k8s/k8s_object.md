# K8s中的对象

Kubernetes 对象是一种“意向表达（Record of Intent）”。一旦创建该对象， Kubernetes 系统将不断工作以确保该对象存在。

### 对象规约（Spec）与状态（Status）

对象 **`spec`（规约）**：描述希望对象所具有的特征，即**期望状态（Desired State）**。

对象 **`status`（状态）**：描述了对象的**当前状态（Current State）**。

对象的 Yaml 定义的构成：

- TypeMeta：Kind，ApiVersion， `inline`对应`kind`和`apiVersion`两个字段；
- ObjectMeta：对应的.metadata；
- Spec：对应的 .spec；
- Status：对应的 .status；



## ObjectMeta

### 名空间

**名字空间（Namespace）** 提供一种机制，将同一集群中的资源划分为相互隔离的组。 同一名字空间内的资源名称要唯一，但跨名字空间时没有这个要求。 

- 名字空间作用域仅针对带有名字空间的对象 （例如 Deployment、Service 等），这种作用域对集群范围的对象 （例如 StorageClass、Node、PersistentVolume 等）不适用。



### 标签和注解

可以使用标签或注解（键值对）将元数据附加到 Kubernetes 对象。

- 标签可以用来选择对象和查找满足某些条件的对象集合，能够支持高效的查询和监听操作。
- 注解不用于标识和选择对象。 注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。 

Map 中的键和值必须是字符串：

- 标签和注解的**键格式**："可选前缀/名称"
  - 前缀必须是 DNS 子域，由点（`.`）分隔的一系列 DNS 标签，总共不超过 253 个字符；
  - 名称 <= 63字符，字母数字开头和结尾，支持`_-.`字符；
- 标签**值格式**：<= 63字符，字母数字开头和结尾，支持`_-.`字符；



### ResourceVersion

> 全局唯一的版本号（`metadata.resourceVersion`），防止基于同一个版本的并发修改的覆盖问题。
>
> - 基于 etcd 的OptimisticPut 和 ModRevision （修改这个 key 时集群的 Revision（全局唯一，递增属性））；

**每个资源对象从创建开始就会有一个版本号，而后每次被修改（不管是 update 还是 patch 修改），版本号都会发生变化。**

- 如果两个用户同时对一个资源对象做 update，不管操作的是对象中同一个字段还是不同字段，都**存在版本控制的机制**确保两个用户的 **update 请求不会发生覆盖**。
- 同一个版本的并发修改，只能有一个成功，另一个再更改时会发现版本号不一致，返回冲突错误（409）；

[官方文档](https://kubernetes.io/docs/reference/using-api/api-concepts/#resource-versions)告诉我们，这个版本号是一个 K8s 的内部机制，用户**不应该假设它是一个数字或者通过比较两个版本号大小**来确定资源对象的新旧，唯一能做的就是**通过比较版本号相等来确定对象是否是同一个版本**（即是否发生了变化）。



### Generation

- 对所有变更请求，除非改变是针对 `.metadata` 或 `.status`，`.metadata.generation` 的取值都会增加。



### Finalizers

> `Finalizers` 是由字符串组成的数组，当 `Finalizers` 字段中存在元素时，相关资源不允许被删除。
>
> - 使用 Finalizers 来控制对象的垃圾回收， 方法是在删除目标资源之前提醒控制器执行特定的清理任务。

每当删除 namespace 或 pod 等一些 Kubernetes 资源时，有时资源状态会卡在 `Terminating`，很长时间无法删除，甚至有时增加 `--force` flag 之后还是无法正常删除。这时就需要 `edit` 该资源，将 `finalizers` 字段设置为 []，之后 Kubernetes 资源就正常删除了。

字段属于 **Kubernetes GC 垃圾收集器**，是一种删除拦截机制，能够让控制器实现**异步的删除前（Pre-delete）回调**，存在于**资源对象的 Meta**中。

对带有 Finalizer 的对象的**第一个删除请求会为其 `metadata.deletionTimestamp` 设置一个值，但不会真的删除对象**。一旦此值被设置，finalizers 列表中的值就**只能**被移除。

当 `metadata.deletionTimestamp` 字段被设置时，负责监测该对象的各个控制器会通过**轮询**对该对象的更新请求来执行它们所要处理的所有 Finalizer。当所有 Finalizer 都被执行过，资源被删除。

- `metadata.deletionGracePeriodSeconds` 的取值控制对更新的轮询周期。

- 每个控制器要负责将其 Finalizer 从列表中去除。每执行完一个就从 `finalizers` 中移除一个，直到 `finalizers` 为空，之后其宿主资源才会被真正的删除。



### Owner References 属主与附属

例如，ReplicaSet 是一组 Pod 的属主，具有属主的对象是属主的附属（Dependent）。附属对象有一个 `metadata.ownerReferences` 字段，用于引用其属主对象。在 Kubernetes 中**不允许跨 namespace 指定属主**。

> delete 策略详细的处理逻辑见[垃圾回收](./garbage_collect.md#级联删除)

**Foreground策略**：先**删除附属对象，再删除属主对象**。将对象的`metadata.finalizers`字段值设置为`foregroundDeletion`，控制器需要主动处理`foregroundDeletion`的finalizers。

**Background策略（默认）**：删除一个对象同时会删除它的附属对象。

**Orphan策略**：不会自动删除它的附属对象，这些残留的依赖被称为原对象的孤儿对象。



## 参考文档

1. [Kubernetes API 概念--ResourceVersion | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/using-api/api-concepts/#resource-versions)
2. 
