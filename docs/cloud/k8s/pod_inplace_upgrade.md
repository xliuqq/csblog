# Pod 原地升级

## 重建升级与原地升级

当我们要升级一个存量 Pod 中的镜像时，这是 *重建升级* 和 *原地升级* 的区别：

**重建升级**时我们要删除旧 Pod、创建新 Pod

- Pod 名字和 uid 发生变化，因为它们是完全不同的两个 Pod 对象（比如 Deployment 升级）
- Pod 名字可能不变、但 uid 变化，因为它们是不同的 Pod 对象，只是复用了同一个名字（比如 StatefulSet 升级）
- Pod 所在 Node 名字发生变化，因为新 Pod 很大可能性是不会调度到之前所在的 Node 节点的
- Pod IP 发生变化，因为新 Pod 很大可能性是不会被分配到之前的 IP 地址的

但是对于**原地升级**，仍然复用同一个 Pod 对象，只是修改它里面的字段。因此：

- 可以避免如 ***调度*、*分配 IP*、*分配、挂载盘*** 等额外的操作和代价
- **更快的镜像拉取**，因为开源复用已有旧镜像的大部分 layer 层，只需要拉取新镜像变化的一些 layer
- 当一个容器在原地升级时，Pod 中的**其他容器不会受到影响**，仍然维持运行

![inplace update comparation pod](.pics/pod_inplace_upgrade/inplace-update-comparation-pod.png)

## K8s 支持的原地升级

K8s 支持的Pod 原地升级仅限于：

- kubectl`patch`  replace 或者 apply 修改 Pod 定义中的 `image` 字段，或者 `metadata`中的 `label`和 `annotation`；
- 对于 deployment 这些管理 pod 的资源，需要找到其 Pod，再进行 patch，**直接对 deployment 修改不是原地升级**；



## [开源实现OpenKruise](https://openkruise.io/zh/docs/core-concepts/inplace-update/)

> 不能脱离 K8s 对 Pod 原地升级的限制，只能修改 image 和 annotation / label 这些字段；

对 OpenKruise 的 Workload 支持`InPlaceIfPossible`，它意味着 Kruise 会尽量对 Pod 采取原地升级，如果不能则退化到重建升级。

以下的改动会被允许执行原地升级：

- 更新 workload 中的 `spec.template.metadata.*`，比如 `labels/annotations`；
  - 如果 env from 这些改动的 labels/anntations，Kruise 会原地升级这些容器来生效新的 env 值；
- 更新 workload 中的 `spec.template.spec.containers[x].image`；

> [SidecarSet](https://openkruise.io/zh/docs/user-manuals/sidecarset) 的原地升级流程和其他 workloads 不太一样，比如它在升级 Pod 之前并不会把 Pod 设置为 not-ready 状态。因此，下文中讨论的内容并不完全适用于 SidecarSet。

以 [CloneSet](https://openkruise.io/zh/docs/user-manuals/cloneset) 为例，其执行流程如下：



![inplace-update-workflow-openkruise](.pics/pod_inplace_upgrade/inplace-update-workflow-openkruise.png)