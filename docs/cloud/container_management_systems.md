# Borg、Omega、K8s：Google 十年三代容器管理系统的设计与思考

> 原文副标题为 ***Lessons learned from three container-management systems over a decade***。
>
> 参考链接：https://arthurchiao.art/blog/borg-omega-k8s-zh

介绍 Google 在设计和使用三代容器管理系统时学到的经验和教训。

## 1. 十年三代容器管理系统

### 1.1 Borg

Google 第一代统一容器管理系统，Borg 当前仍然是 Google 内部主要的容器管理系统（巨大的规模、出色的功能和极其的健壮性）

- 管理 **long-running service** 也可以管理 **batch job**；
- **Linux 内核中新出现的容器技术**（Google 给 Linux 容器技术贡献了大量代码），实现**延迟敏感型应用**和 **CPU 密集型批处理任务**之间的更好隔离；
- 内部形成的生态是一堆**异构、自发的工具和系统**（而非一个有设计的体系），交互涉及多种语言和配置；

### 1.2 Omega

使 Borg 的生态系统更加符合软件工程规范，开发了 Omega，继承许多已经在 Borg 中经过验证的成功设计，但又是**完全从头开始开发**， 以便**架构更加整洁一致**

- 将**集群状态**存储在一个基于 Paxos 的中心式面向事务 store（数据存储）内；
- **控制平面组件**（例如调度器）都可以**直接访问这个 store**；
- 用**乐观并发控制**来处理偶发的访问冲突。

**Borg master 的功能拆分为了几个彼此交互的组件**， 而不再是一个单体的、中心式的 master，修改和迭代更加方便。

### 1.3 Kubernetes

开发这套系统的背景：

- 全球越来越多的开发者也开始对 Linux 容器感兴趣，
- Google 已经把公有云基础设施作为一门业务在卖，且在持续增长；

Kubernetes 是开源的，不是 Google 内部系统。相比 Omega 的改进

- 与 Omega 类似，k8s 的核心也是一个**共享持久数据仓库**（store）， 几个组件会监听这个 store 里的 object 变化；
- Omega 将自己的 store 直接暴露给了受信任的控制平面组件，但 k8s 中的状态 **只能通过一个 domain-specific REST API 访问**，这个 API 会执行 `higher-level versioning, validation, semantics, policy` 等操作，支持多种不同类型的客户端；
- 更重要的是，k8s 在设计时就**非常注重应用开发者的体验**： 首要设计目标就是在享受容器带来的资源利用率提升的同时，**让部署和管理复杂分布式系统更简单**

## 2. 底层 Linux 内核容器技术

### 2.1 从 `chroot` 到 cgroups

- 最初的容器只是提供了 **root file system 的隔离能力** （通过 **`chroot`**）
- FreeBSD jails 将这个理念扩展到了对其他 namespaces（例如 PID）的隔离
- Linux control groups (**`cgroups`**) 吸收了这些理念，成为集大成者

### 2.2 资源隔离

Borg 能利用容器实现**延迟敏感型应用**和**CPU 密集型批处理任务**的混布（co-locate）， 从而提升资源利用率：

- 业务通常申请的资源量要大于他们实际需要的资源量，通过混部就能把这些资源充分利用起来，给批处理任务使用；
- 强大的内核**资源隔离技术**， 就能避免这两种类型任务的互相干扰；

这种隔离并未达到完美的程度：容器无法避免那些不受内核管理的资源的干扰，例如**三级缓存（L3 cache）、 内存带宽**；此外，还需要对容器加一个安全层（例如虚拟机）才能避免公有云上各种各样的恶意攻击。

### 2.3 容器镜像

现代容器已经不仅仅是一种隔离机制了：还包括镜像 —— 将应用运行所需的所有文件打包成一个镜像。

在 Google，用 MPM (Midas Package Manager) 来构建和部署容器镜像。 隔离机制和 MPM packages 的关系，就像是 Docker daemon 和 Docker image registry 的关系。



## 3. 面向应用的基础设施

从原来的**面向机器**（machine oriented） 转向了**面向应用**（application oriented）

### 3.1 从“面向机器”到“面向应用”的转变

容器化使数据中心的观念从原来的**面向机器**（machine oriented） 转向了**面向应用**（application oriented），

- **容器封装了应用环境**（application environment）， 向应用开发者和部署基础设施**屏蔽了大量的操作系统和机器细节**，
- 每个设计良好的**容器和容器镜像都对应的是单个应用**，因此 **管理容器其实就是在管理应用，而不再是管理机器**。

Management API 的这种从面向机器到面向应用的转变，显著提升了应用的部署效率和问题排查能力。

### 3.2 应用环境

#### 资源隔离 + 容器镜像：解耦应用和运行环境

资源隔离能力与容器镜像相结合，创造了一个全新的抽象：

- 内核 cgroup、chroot、namespace 等基础设施的最初目的是**保护应用免受 noisy、nosey、messy neighbors 的干扰**。
- 而这些技术**与容器镜像相结合**，创建了一个**新的抽象**， **将应用与它所运行的（异构）操作系统隔离开来**。

抽象能成功的关键是一个*hermetic*（封闭的，不受外界影响的）容器镜像：

- 这个镜像能封装一个应用的几乎所有依赖（文件、函数库等等）；
- 那**唯一剩下的外部依赖就是 Linux 系统调用接口**：有限的接口极大提升了镜像的可移植性，但并非完美
  - 应用仍然可能*通过 socket option、`/proc`、`ioctl` 参数等等产生很大的暴露面*，希望 [Open Container Initiative](https://www.opencontainers.org/) 等工作可以进一步明确容器抽象的 surface area。

#### 容器镜像实现方式

Borg container image 并没有做到完全独立：所有应用共享一个所谓的 *base image*，基础镜像是安装在每个 node 上的，而非打到每个镜像里。升级基础镜像时会影响已经在运行的容器（应用），偶尔会导致故障。

Docker 和 ACI 现代容器镜像**消除了隐藏的 host OS 依赖**， 明确要求用户在容器间共享镜像时，必须显式指定这种依赖关系。

### 3.3 容器作为基本管理单元

围绕容器而非机器构建 management API，将数据中心的核心从机器转移到了应用

1. 应用开发者和应用运维团队**无需再关心机器和操作系统**等底层细节；
2. 基础设施团队**引入新硬件和升级操作系统更加灵活**， 可以最大限度减少对线上应用和应用开发者的影响；
3. 将收集到的 telemetry 数据（例如 CPU、memory usage 等 metrics）关联到应用而非机器， **显著提升了应用监控和可观测性**，尤其是在*垂直扩容、 机器故障或主动运维等需要迁移应用*的场景。

#### 3.3.1 通用 API 和自愈能力

容器能提供一些通**用的 API 注册机制**，使管理系统和应用之间**无需知道彼此的实现细节**就能交换有用信息

- `health check`：HTTP endpoint 或 shell 命令（到容器内执行）。当检测到一个不健康的应用时， 就会自动终止或重启对应的容器。这种**自愈能力**（self-healing）是构建可靠分布式系统的最重要基石之一。

#### 3.3.2 用 annotation 描述应用结构信息

容器还能提供或展示其他一些信息。例如，

- Borg 应用可以提供一个字符串类型的状态消息，这个字段可以动态更新；
- K8s 提供了 key-value **`annotation`**， 存储在每个 object metadata 中，可以用来传递**应用结构（application structure）信息**。 

#### 3.3.3 应用维度 metrics 聚合

容器还能用其他方式提供面向应用的监控：例如， cgroups 提供了应用的 resource-utilization 数据。

**基于这些监控数据就能开发一些通用工具**，例如 **auto-scaler 和 cAdvisor**

- 应用收敛到了容器内，因此就无需在宿主机上分发信号到不同应用
- 更简单、更健壮， 也更容易实现细粒度的 metrics/logs 控制

面向应用的转变在管理基础设施（management infrastructure）中产生涟漪效应

- load balancer 不再针对 machine 转发流量，而是**针对 application instance** 转发流量；
- Log 自带应用信息，因此很容易收集和按应用维度（而不是机器维度）聚合， 从而更容易看出**应用层面的故障**；
- 实例在**编排系统中的 identity 和用户期望的应用维度 identity 能够对应**起来， 因此更容易构建、管理和调试应用。

#### 3.3.4 单实例多容器（pod vs. container）

使用嵌套容器，对于一个应用：

- **外层容器提供一个资源池**；在 Borg 中成为 `alloc`，在 K8s 中成为 `pod`；
- 内层容器们部署和隔离具体服务。

K8s 就统一规定**应用容器必须运行在 pod** 内，即使一个 pod 内只有一个容器。 常见方式是一个 pod hold 一个复杂应用的实例。

- 应用的主体作为一个 pod 内的容器。
- 其他辅助功能（例如 log rotate、click-log offloading）作为独立容器。

相比于把所有功能打到一个二进制文件，这种方式能让不同团队开发和管理不同功能，好处：

- 健壮：例如，应用即使出了问题，log offloading 功能还能继续工作；
- 可组合性：添加新的辅助服务很容易，因为操作都是在它自己的 container 内完成的；
- 细粒度资源隔离：每个容器都有自己的资源限额，比如 logging 服务不会占用主应用的资源。

### 3.4 编排是开始，不是结束

#### 3.4.1 自发和野蛮生长的 Borg 软件生态

Borg 本身只是**开发和管理可靠分布式系统的开始**， 各团队根据自身需求开发出的围绕 Borg 的不同系统与 Borg 本身一样重要

- 服务命名（naming）和**服务发现**（Borg Name Service, BNS）；自动扩缩容；Workflow工具；监控工具

#### 3.4.2 避免野蛮生长：K8s 统一 API（Object Metadata、Spec、Status）

K8s 尝试通过引入一致 API 的方式来降低这里的复杂度。例如，**每个 K8s 对象都有三个基本字段**：**`Object Metadata`**，**`Spec`**，**`Status`**。

统一 API 提供了几方面好处：

- 学习更加简单，因为所有 object 都遵循同一套规范和模板；
- 编写适用于所有 object 的通用工具也更简单；
- 用户体验更加一致。

#### 3.4.3 K8s API 扩展性和一致性

K8s 还进行了扩展，**支持用户动态注册他们自己的 API**， 这些 API 和它内置的核心 API 使用相同的方式工作。API 组件的解耦考虑意味着上层服务可以共享相同的基础构建模块：如**replica controller 和 horizontal auto-scaling (HPA) 的解耦**。

- Replication controller **确保**给定角色（例如，”front end”）的 **pod 副本数量符合预期**；
- Autoscaler 这利用这种能力，简单地**调整期望的 pod 数量**，而无需关心这些 pod 是如何创建或删除的。

#### 3.4.4 Reconcile 机制

通过让不同 k8s 组件使用同一套设计模式来实现一致性：`reconciliation controller loop`

- **`reconcile`**（调谐）：首先对观测到的当前状态（“当前能找到的这种 pod 的数量”）和期望状态（“label-selector 应该选中的 pod 数量”）进行比较；如果当前状态和期望状态不一致，则执行相应的行动 （例如扩容 2 个新实例）来使当前状态与期望相符；
- 所有操作都是**基于观察**（observation）**而非状态机**， 因此 reconcile 机制**非常健壮**：controller 重启之后仍能接着之前状态继续工作；

#### 3.4.5 舞蹈编排（choreography）vs. 管弦乐编排（orchestration）

**choreography（舞蹈编排）**的一个例子 —— 通过多个独立和自治的实体之间的协作（collaborate）实现最终希望达到的状态。

> 舞蹈编排：场上**没有**指挥老师，每个跳舞的人都是独立个体，大家共同协作完成一次表演。 代表**分布式**、非命令式。

管弦乐编排**中心式编排系统**（centralized orchestration system），后者在初期很容易设计和开发， 但随着时间推移会变得脆弱和死板，尤其在*有状态变化或发生预期外的错误*时

> 管弦乐编排：场上有一个指挥家，每个演奏乐器的人都是根据指挥家的命令完成演奏。 代表**集中式**、命令式。



## 4. 避坑指南

### 4.1 创建 Pod 时应该分配唯一 IP，而不是唯一端口（port）

在 Borg 中，容器没有独立 IP，**所有容器共享 node 的 IP**。 Borg 只能在调度时，**给每个容器分配唯一的 port**。 当一个容器漂移到另一台 node 时，会获得一个新的 port（容器原地重启也可能会分到新 port）

- 类似 **DNS**（运行在 `53` 端口）这样的传统服务，只能用一些**内部魔改的版本**；
- 客户端无法提前知道一个 service 的端口，只有在 service 创建好之后再告诉它们；
- URL 中不能包含 port（容器重启 port 可能就变了，导致 URL 无效），必须引入一些 name-based redirection 机制；
- 依赖 IP 地址的工具都必须做一些修改，以便能处理 `IP:port`。

设计 k8s 时，我们决定**给每个 pod 分配一个 IP**，

- 这样就实现了**网络身份**（IP）与**应用身份**（实例）的一一对应；
- 避免了前面提到的魔改 DNS 等服务的问题，应用可以随意使用 well-known ports（例如，HTTP 80）；
- 现有的网络工具（例如网络隔离、带宽控制）也无需做修改，直接可以用；

此外，所有公有云平台都提供 IP-per-pod 的底层能力；在 bare metal 环境中，可以使用 SDN overlay 或 L3 routing 来实现每个 node 上多个 IP 地址。

### 4.2容器索引不要用数字 index，用 labels

用户一旦习惯了容器开发方式，马上就会创建一大堆容器出来， 因此接下来的一个需求就是如何对这些容器进行**分组和管理**。

#### Borg 基于 index 的容器索引设计

Borg 提供了 *jobs* 来对容器名字相同的 *tasks* 进行分组。

- 每个 job 由一个或多个完全相同的 task 组成，用向量（vector）方式组织，从 index=0 开始索引。

这种方式非常简单直接，很好用，但随着时间推移，我们越来越发觉它比较死板。

- 一个 task 挂掉后在另一台 node 上被创建出来时，task vector 中对应这个 task 的 slot 必须做双份的事情： 识别出新的 task；在需要 debug 时也能指向老的 task；
- 当 task vector 中间某个 task 正常退出之后，vector 会留下空洞；
- vector 也很难支持跨 Borg cluster 的 job；
- 如果**用户基于 task index 来设计 sharding**，那 Borg 的重启策略就会导致数据不可用，因为它都是按 task 顺序重启的；

#### K8s 基于 label 的容器索引设计

k8s 主要使用 *labels* 来识别一组容器（groups of containers）

- Label 是 key/value pair，包含了可以用来鉴别这个 object 的信息； 
- Label 可以动态添加、删除和修改，可以通过工具或用户手动操作；
- **不同团队可以管理各自的 label**，基本可以做到不重叠；

通过 **`label selector`** （例如，`stage==production && role==frontend`）来选中一组 objects

- 组可以重合，也就是一个 object 可能出现在多个 label selector 筛选出的结果中，因此这种基于 label 的方式更加灵活；
- label selector 是动态查询语句，也就是说只要用户有需要，他们随时可以按自己的需求编写新的查询语句（selector）；
- Label selector 是 k8s 的 grouping mechanism，也定义了所有管理操作的范围（多个 objects）。

Labels 和 label selectors 提供了一种通用机制， **既保留了 Borg 基于 index 索引的能力，又获得了上面介绍的灵活性**。

### 4.3 Ownership 设计要格外小心

在 Borg 中，task 并不是独立于 job 的存在：

- 创建一个 job 会创建相应的 tasks，这些 tasks 永远与这个 job 绑定；
- 删除这个 jobs 时会删除它所有的 tasks。

当这个 vector index 的 grouping 机制抽象无法满足某些场景（如DaemonSet 需要将一个 pod 在每个 node 上都起一个实例）时， 用户必须开发一些 workaround。

Kubernetes 中，那些 **pod-lifecycle 管理组件** —— 例如 replication controller —— **通过 label selector 来判断哪些 pod 归自己管**；

- 多个 controller 可能会选中同一个 pod，认为这个 pod 都应该归自己管， 这种冲突理应在配置层面解决。

label灵活性的好处：controller 和 pod 分离意味着**可以 "orphan" 和 "adopt" container**

- service， 如果其中一个 pod 有问题了，那只需要把相应的 label 从这个 pod 上去掉， k8s service 就不会再将流量转发给这个 pod
  - pod 不再接生产流量，但仍然活在线上，因此就可以对它进行 debug 
  - 负责管理这些 pod 的 replication controller 就会立即再创建一个新的 pod 出来接流量

### 4.4 不要暴露原始状态

> Borg、Omega 和 k8s 的一个**核心区别**是它们的 **API 架构**。

**Borgmaster 是一个单体组件，理解每个 API 操作的语义**

- 包含集群管理逻辑，例如 job/task/machine 的状态机；
- 运行基于 Paxos 的 replicated storage system，用来存储 master 的状态。

**Omega 除了 store 之外没有中心式组件**

- **store 存储了 passive 状态信息，执行乐观并发控制**；
- **所有逻辑和语义都下放到了操作 store 的 client 上**，后者会直接读写 store 内容；
- 每个 Omega 组件都使用了同一套 client-side library 来与 store 交互， 这个 liabrary 做的事情包括 packing/unpacking of data structures、retries、 enforce semantic consistency。

**k8s 在 Omega 的分布式架构和 Borg 的中心式架构之间做了一个折中**

- 系统级别的规范、策略、数据转换等方面还是**集中式**： **store 前面加了一层集中式的 API server**，屏蔽掉所有 store 实现细节， 提供 object validation、defaulting 和 versioning 服务。
- 与 Omega 类似，**客户端组件都彼此独立**，可以独立开发、升级（在开源场景尤其重要）， 但各组件都要经过 apiserver 这个中心式服务的语义、规范和策略



## 5 开放问题讨论

### 5.1 应用配置管理

**应用配置**，即如何把应用的参数在创建容器时传给它， 而不是 hard-code。

承认 **`programmatic configuration`** 的不可避免性， 在计算和数据之间维护一条清晰边界。 **表示数据的语言应该是简单、data-only 的格式**，例如 JSON or YAML， 而针对这些数据的计算和修改应该在一门真正的编程语言中完成，后者有 完善的语义和配套工具。

### 5.2 依赖管理

上线一个新服务通常也意味着需要上线一系列相关的服务（监控、存储、CI/CD 等等）

- **自动初始化依赖（dependencies）并不是仅仅启动一个新实例**

一种可能的解决方式是：每个应用显式声明它依赖的服务，基础设施层禁止它访问除此之外的所有服务 （我们的构建系统中，编译器依赖就是这么做的）。 好处就是基础设施能做一些有益的工作，例如自动化设置、认证和连接性。

- **表达、分析和使用系统依赖会导致系统的复杂性升高**，并没有任何一个主流的容器管理系统做了这个事情