# 调度系统



## 工作流调度

- **DolphinScheduler（DS）：工作流任务调度引擎**，支持多种任务类型；

- **Argo Workflow**：**工作流编排调度引擎**，只运行在 K8s 集群，不支持 YARN、物理机、本地 Worker；



## 资源调度

### SLURM（超算）



### YARN（大数据）

**YARN：大数据资源调度框架**，管**集群计算资源分配、容器资源隔离**，大数据层资源调度；



### K8S（云原生）

**K8s：容器编排资源调度平台**，管**容器生命周期、机器资源、服务编排**，底层基础设施调度。

![compare](pics/README/volcano_kueue_yunikorn_compare.png)

Yunikorn：YARN 风格**无限层级树形队列**。替换 `Default scheduler` 的**底层节点调度器，不改变上层 Job/Pod 定义**。

Kueue：**作业准入 / 配额排队层**，复用 `Default scheduler`，不做节点分配。

**Volcano**：调度器 + 作业控制器（PodGroup/Gang） 一体化，完全替换默认调度器

