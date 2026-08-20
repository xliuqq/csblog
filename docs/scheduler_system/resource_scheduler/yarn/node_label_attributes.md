# 节点标签与属性

## 节点标签

> https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/NodeLabel.html
>
> 用于集群分组

通过标签将集群分区

- 未配置标签的节点，会被划分到 default 分区；
- 队列可以占据不同的分区的不同百分比，剩余的会被 default 队列占据；
- 提交作业到队列上时，如果队列可用多个分区，可以指定使用特定的分区；

节点标签策略

- **Exclusive**：容器将分配给节点**分区完全匹配**的节点；
- **Non-exclusive**：**分区非独占，则会向DEFAULT分区共享资源**，即Container申请DEFAULT分区也可以使用分区A的空闲资源
  - 如果 DEFAULT 分区使用分区A的空闲资源后，分区A有作业提交，DEFAULT分区使用的资源不会退还； 



### 作业提交时的分区限制

对于GPU作业，通过spark封装提交到yarn上，并且**通过节点标签的限制，将executor限制到GPU分区上运行**。

同一个队列，配CPU和GPU分区的容量，如何**限制CPU作业运行到GPU分区上**？

> Specify required node label in resource request, it will only be allocated when node has the same label. **If no node label requirement specified, such Resource Request will only be allocated on nodes belong to DEFAULT partition.**

通过官方说明，除了AI作业（GPU label）外，其它作业可以**通过不指定标签，将作业直接限制在default分区**。

- 通过`yarn.scheduler.capacity.<queue-path>.default-node-label-expression`，可以修改默认时不设置分区时的默认分区；



### AM 设置节点标签

```scala
val amRequest = Records.newRecord(classOf[ResourceRequest])
amRequest.setResourceName(ResourceRequest.ANY)
amRequest.setPriority(Priority.newInstance(0))
amRequest.setCapability(capability)
amRequest.setNumContainers(1)
amRequest.setNodeLabelExpression(expr)
appContext.setAMContainerResourceRequest(amRequest)
```

### Container 设置节点标签

```java
new ContainerRequest(..., nodeLabelExpresssion)
```



### API

通过 CLI 或者 Client API 新增或修改分区时，需要注意**分区被绑定的所有节点（包括默认default队列）的总量**为 100%

- 新增分区时，
  - 设置先设置 root 和 root.default 对该分区的容量为100%；
  - 新增队列，容量为0；
  - 修改新增队列和default队列对分区的容量；



## 节点属性

> https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/NodeAttributes.html
>
> 用于 Placement Constraint，队列不需要做任何配置。

特性：

- 硬性限制，不像Locality具备Relax；
- 可持久化，RM运行时可更新；

节点的属性来源支持两种方式：

- Centralised：通过 RM 的 CLI 或者 RPC（REST暂不支持）；
- Distribtued：每个NM配置Provider（scritp或者config）；

### 调度策略

`yarn.resourcemanager.placement-constraints.handler`：默认 disabled

- **placement-processor**：先确定 placement，然后scheduler再调度（a pre-processing step before the capacity or the fair scheduler is called）
  - 支持所有限制类型，`affinity, anti-affinity, cardinality`；
  - 一次处理多个containers，容量/公平调度都支持；
  - 不考虑应用内的 task 优先级（一个AM可以申请不同优先级的Container），因为这些优先级可能与放置约束相冲突；
- **scheduler**：仅支持容量调度，在调度的同时考虑 placement；
  - 仅支持 anti-affinity 限制；
  - 对队列（按利用率、优先级排序）、应用程序（按FIFO/公平性/优先级排序）和同一应用程序内的任务（优先级）遵循相同的排序规则

### Placement语法

> https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/PlacementConstraints.html
>
> constraints can refer to allocation tags, as well as node labels and node attributes.

> ```
> SingleConstraint  => "IN",Scope,TargetTag | "NOTIN",Scope,TargetTag | "CARDINALITY",Scope,TargetTag,MinCard,MaxCard | NodeAttributeConstraintExpr
> NodeAttributeConstraintExpr => NodeAttributeName=Value, NodeAttributeName!=Value
> Scope                       => "NODE" | "RACK"
> ```

allocation tags：attached to allocations and not to nodes；

- 用于 IN，NOTIN 和 CARDINALITY；
- APP 申请Container时，配置的 tags，比如可以对HBase Master Container 打 hbase-m的tag，Region Server打 hbase-rs 的标签；
- 可作用于**应用程序内部或者应用程序间**，默认是应用程序内部的判断；
  - `self/${tag}, not-self/${tag}, all/${tag}`
  - `app-id/${applicationID}/${allocationTag}`，`app-tag/application_tag_name/${allocationTag}`

node attributes：KV 的形式



### 应用配置

#### AM 不支持配置节点属性

> https://issues.apache.org/jira/browse/YARN-10065

#### Container 设置节点属性

- 配置使用`placement-processor` handler，针对所有的 container；

```scala
val amClient = AMRMClient.createAMRMClient()
...
amClient.registerApplicationMaster(..., placementConstraints)
```

- 配置使用 `scheduler`handler，除了上面之外，可以针对每个 container 设置（优先级最高，覆盖前者），使用 SchedulingRequest 而不是 ContainerRequest

```java
//requests for 1 container on nodes expression : AND(python!=3:java=1.8)
SchedulingRequest schedulingRequest =
    SchedulingRequest.newBuilder().executionType(
        ExecutionTypeRequest.newInstance(ExecutionType.GUARANTEED))
        .allocationRequestId(10L).priority(Priority.newInstance(1))
        .placementConstraintExpression(
            PlacementConstraints.and(
                PlacementConstraints
                    .targetNodeAttribute(PlacementConstraints.NODE,
                        NodeAttributeOpCode.NE,
                        PlacementConstraints.PlacementTargets
                            .nodeAttribute("python", "3")),
                PlacementConstraints
                    .targetNodeAttribute(PlacementConstraints.NODE,
                        NodeAttributeOpCode.EQ,
                        PlacementConstraints.PlacementTargets
                            .nodeAttribute("java", "1.8")))
                .build()).resourceSizing(
        ResourceSizing.newInstance(1, Resource.newInstance(1024, 1)))
        .build();
```

