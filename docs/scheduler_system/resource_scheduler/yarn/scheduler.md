# 调度策略

## 原理

队列内应用的排序顺序，

ActiveApp的OrderPolicy：默认是FIFO（FifoOrderingPolicy），可以通过`yarn.scheduler.capacity.<queue-name>.ordering-policy`自定义。

PendingApp的OrderPolicy：固定的 FifoOrderingPolicyForPendingApps。

## 容量调度

**CapacityScheduler**：允许多个租户安全地共享一个大型集群，以便在分配的容量限制下及时地分配其应用程序的资源。

CapacityScheduler对来自单个用户和队列的已初始化和挂起的应用程序提供了限制，以确保集群的公平性和稳定性。

### 特性

- **分层队列**：支持队列层次结构，以确保在**允许其他队列使用空闲资源之前，在组织的子队列之间共享资源**，从而提供更多的控制和可预测性；
- **容量保证**：队列被分配到网格容量的一小部分，这意味着将有**一定的资源容量**供其使用。**所有提交到队列的应用程序都可以访问分配给队列的容量**。管理员可以配置分配给每个**队列的容量的软限制和可选的硬限制**。
- **安全性**：每个**队列都有严格的acl**，用于控制哪些用户可以向单个队列提交应用程序。另外，还有一些安全保护措施可以确保用户不能查看和/或修改来自其他用户的应用程序。此外，还**支持每个队列和系统管理员角色**。
- **弹性**：可以**将空闲资源分配给超出其容量的任何队列**。当未来某个时间点上运行的容量不足的队列对这些资源有需求时，随着在这些资源上调度的任务完成，它们将**被分配给容量不足的队列上的应用程序(也支持抢占)**。这确保了资源以可预测的、弹性的方式可用于队列，从而防止了集群中资源的人工竖井（artificial silos），从而有助于利用资源。
- **多租户**：提供了一组全面的限制，以**防止单个应用程序、用户和队列作为一个整体垄断队列或集群的资源**，以确保集群不会过载。
- **可操作性**：
  - 运行时配置：管理员可以在**运行时以安全的方式更改队列定义和属性**(如容量、acl)，以最大限度地减少对用户的干扰。此外，还提供了一个控制台，供用户和管理员查看系统中各种队列的当前资源分配情况。**管理员可以在运行时添加其他队列，但不能在运行时删除队列，除非队列被停止**，并且没有挂起/运行的应用程序。
  - 释放应用程序：**管理员可以在运行时停止队列**，以确保在现有应用程序运行到完成时，不会提交新的应用程序。如果队列处于停止状态，则无法将新应用程序提交给队列本身或其任何子队列。现有的应用程序将继续完成，因此可以优雅地释放队列。**管理员还可以启动已停止的队列**。
- **基于资源的调度**：支持**资源密集型应用程序**，其中应用程序中可以选择性地指定比默认值更高的资源需求，从而适应不同资源需求的应用程序。
- **优先级调度**：以不同的优先级提交和调度应用程序。较高的整数值表示应用程序的优先级较高。目前，**应用程序优先级仅支持FIFO排序策略**。
- **基于默认或用户定义的放置规则的队列映射接口**：允许**用户基于某些默认放置规则将作业映射到特定队列**。例如，根据用户和组或应用程序名称。用户也可以定义自己的放置规则（Hadoop 3.x）。
- **绝对的资源配置（Hadoop 3.x）**：**管理员可以为队列指定绝对资源**，而不是提供基于百分比的值。
- **Leaf Queue的动态自动创建和管理（Hadoop 3.x）**：支持**自动创建叶队列和队列映射**，队列映射目前支持**基于用户组的队列映射**，用于将应用程序放置到队列中。调度器还支持基于父队列上配置的策略对这些队列进行容量管理。

### 配置

`yarn.resourcemanager.scheduler.class`：默认，`org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler`

`etc/hadoop/capacity-scheduler.xml`是容量调度的配置文件。

#### 设置队列

`root`是容量调度预先定义好的队列，系统中所有的队列都是root队列的子队列。

*queue path*用来表示分层队列，是队列层次的全路径，以`root`开始，`.`作为分隔。

`yarn.scheduler.capacity.root.queues`定义root的子队列，`yarn.scheduler.capacity.<queue-path>.queues`表示特定队列的子队列。

除非另有说明，子队列不会直接从父队列继承属性。

```xml
<property>
  <name>yarn.scheduler.capacity.root.queues</name>
  <value>a,b,c</value>
  <description>root queue has three sub queue a, b and c.</description>
</property>

<property>
  <name>yarn.scheduler.capacity.root.a.queues</name>
  <value>a1,a2</value>
  <description>queue a has sub queues a1 and a2 </description>
</property>

<property>
  <name>yarn.scheduler.capacity.root.b.queues</name>
  <value>b1,b2,b3</value>
  <description>queue b has sub queues b1 b2 and b3</description>
</property>
```

#### 队列属性

- ##### 资源分配

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.<queue-path>.capacity`              | **队列的容量**，float型百分比或者绝对的容量。每一层的所有队列的容量和必须为100。但如果配置绝对资源容量，则子队列的绝对资源容量和可以小于父队列的绝对资源容量。如果有空闲资源，队列中的应用程序消耗的资源可能超过队列的容量，从而提供灵活性。 |
| `yarn.scheduler.capacity.<queue-path>.maximum-capacity`      | **最大队列的容量**百分比或者列队的绝对资源最大容量。限制队列应用的灵活性。-1表示在2.x表示禁用，在3.x表示100% |
| `yarn.scheduler.capacity.<queue-path>.minimum-user-limit-percent` | 如果有资源需求，每个队列在任何给定时间都对分配给用户的资源百分比施加限制。**用户限制可以在最小值和最大值之间变化**。**前者(最小值)设置为此属性值，后者(最大值)取决于提交应用程序的用户数量**。例如，假设这个属性的值是25。如果两个用户向一个队列提交了应用程序，那么任何一个用户都不能使用超过50%的队列资源。如果第三个用户提交应用程序，任何一个用户都不能使用超过33%的队列资源。对于4个或4个以上的用户，任何用户都不能使用超过25%的队列资源。值为100意味着没有施加任何用户限制。默认值是100。值指定为整数。 |
| `yarn.scheduler.capacity.<queue-path>.user-limit-factor`     | **队列容量的倍数，可配置为允许单个用户获取更多资源**。默认情况下，这个值被设置为1，这确保了无论集群有多空闲，单个用户都不会占用超过队列配置的容量。值被指定为一个浮点数。 |
| `yarn.scheduler.capacity.<queue-path>.maximum-allocation-mb` | 资源管理器上**分配给每个容器请求的每个队列的最大内存限制**。此设置覆盖集群配置`yarn.scheduler.maximum-allocation-mb`。这个值必须小于或等于集群的最大值。 |
| `yarn.scheduler.capacity.<queue-path>.maximum-allocation-vcores` | 资源管理器上**分配给每个容器请求的每个队列虚拟核的最大限制**。此设置覆盖集群配置`yarn.scheduler.maximum-allocation-vcores`。这个值必须小于或等于集群的最大值。 |
| `yarn.scheduler.capacity.<queue-path>.user-settings.<user-name>.weight` | 在为**队列中的用户计算用户限制资源值**时使用此浮点值。这个值将使每个用户的权重大于或小于队列中的其他用户。例如，如果用户A在队列中接收到的资源比用户B和C多50%，那么对于用户A，这个属性将被设置为1.5。用户B和C默认为1.0。 |

- ##### Running和Pending的应用限制

  容量调度器支持以下参数来控制运行和挂起的应用程序:

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.maximum-applications` / `yarn.scheduler.capacity.<queue-path>.maximum-applications` | Integer。**系统中可以同时活动、运行和挂起的应用程序的最大数量**。每个队列上的限制与它们的队列容量和用户限制成正比。这是一个硬限制，当达到这个限制时提交的任何申请都将被拒绝。默认是10000。可以设置所有队列或单个队列。 |
| `yarn.scheduler.capacity.maximum-am-resource-percent` / `yarn.scheduler.capacity.<queue-path>.maximum-am-resource-percent` | **集群中可用于运行am的最大资源百分比**—**控制并发活动应用程序的数量**。每个队列上的限制与它们的队列容量和用户限制成正比。指定为浮点数-即0.5 = 50%。默认是10%。 |

- ##### 队列管理和权限

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.<queue-path>.state`                 | **队列的状态**，可以是 `RUNNING` or `STOPPED`。 如果一个队列处于“停止”状态，新的应用程序不能提交到“自身”或“其子队列”。现有的应用程序继续完成，因此可以优雅地“耗尽”队列。 |
| `yarn.scheduler.capacity.root.<queue-path>.acl_submit_applications` | ACL，用于**控制谁可以向给定队列提交应用程序**。如果给定用户/组对给定队列或层次结构中的一个父队列具有必要的acl，则可以提交应用程序。如果未指定，此属性的acl将从父队列继承。 |
| `yarn.scheduler.capacity.root.<queue-path>.acl_administer_queue` | ACL，用于**控制谁可以管理给定队列上的应用程序**。如果给定用户/组对给定队列或层次结构中的父队列具有必要的acl，则可以管理应用程序。如果未指定，此属性的acl将从父队列继承。 |

Note：ACL的形式是*user1,user2 space group1,group2*。通配符\*表示任意。空格表示没有。默认*root* queue是*。

- ##### 基于用户或组、应用程序名称或用户定义的放置规则的队列映射。

| Property                                                 | Description                                                  |
| :------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.queue-mappings`                 | 指定**用户或组到特定队列的映射**。将单个用户或用户列表映射到队列。语法: `[u or g]:[name]:[queue_name][,next_mapping]*`。*u or g* 表示映射是针对用户还是针对组。*name* 表示用户名或组名。 要指定提交应用程序的用户，可以使用 %user。*queue_name* 表示应用程序必须为其映射的队列名称。若要指定队列名与用户名相同，可以使用%user。若要指定与用户所属的主组名称相同的队列名称，可以使用%primary_group。 |
| `yarn.scheduler.queue-placement-rules.app-name`          | 指定application_name到特定队列的映射。可以将单个应用程序或应用程序列表映射到队列。语法: `[app_name]:[queue_name][,next_mapping]*`. *app_name* 表示要进行映射的应用程序名称。 *queue_name* 表示应用程序必须为其映射的队列名称。要将当前应用程序的名称指定为app_name，可以使用%application。 |
| `yarn.scheduler.capacity.queue-mappings-override.enable` | 指定**是否可以覆盖用户指定的队列**。这是一个布尔值，默认值为false。 |

示例：

```xml
<property>
   <name>yarn.scheduler.capacity.queue-mappings</name>
   <value>u:user1:queue1,g:group1:queue2,u:%user:%user,u:user2:%primary_group</value>
   <description>
     Here, <user1> is mapped to <queue1>, <group1> is mapped to <queue2>, 
     maps users to queues with the same name as user, <user2> is mapped 
     to queue name same as <primary group> respectively. The mappings will be 
     evaluated from left to right, and the first valid mapping will be used.
   </description>
 </property>

  <property>
    <name>yarn.scheduler.queue-placement-rules.app-name</name>
    <value>appName1:queue1,%application:%application</value>
    <description>
      Here, <appName1> is mapped to <queue1>, maps applications to queues with
      the same name as application respectively. The mappings will be
      evaluated from left to right, and the first valid mapping will be used.
    </description>
  </property>
```

原理：

`yarn.scheduler.queue-placement-rules`默认配置是`user-group`，使用`UserGroupMappingPlacementRule`，指定用户提交作业的作业放置规则。

当前内置支持`user-group`和`app-name`，也支持自定义的类（实现`PlacementRule`）

- ##### 应用程序的队列生存期

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.<queue-path>.maximum-application-lifetime` | 提交**到队列的应用程序的最大生存期(以秒为单位)**。任何小于或等于0的值都被认为是禁用的。如果配置为正值，那么提交到此队列的任何应用程序都将在超过配置的生存期后终止。用户还可以在应用程序提交上下文中指定每个应用程序的生存期。但是**如果用户生命周期超过队列最大生命周期，则会被覆盖**。注意:配置过低的值会导致更快地终止应用程序。该特性仅适用于叶队列。 |
| `yarn.scheduler.capacity.root.<queue-path>.default-application-lifetime` | **提交到队列的应用程序的默认生存期**，以秒为单位。任何小于或等于0的值都被认为是禁用的。**如果用户没有提交具有生存期值的应用程序，那么将取此值**。注意:默认生存期不能超过最大生存期。该特性仅适用于叶队列。 |

#### 应用优先级

**应用程序优先级只与FIFO排序策略一起工作**。默认的排序策略是FIFO。

应用程序的默认优先级可以是集群级和队列级。**当应用程序移动到不同队列时，应用程序的优先级不会改变。**

| Property                                                     | Description                        |
| :----------------------------------------------------------- | :--------------------------------- |
| `yarn.cluster.max-application-priority`                      | 定义集群中的最大应用程序优先级。   |
| `yarn.scheduler.capacity.root.<leaf-queue-path>.default-application-priority` | 定义叶队列中的默认应用程序优先级。 |

- **集群级优先级**：任何优先级大于集群最大优先级的提交应用程序的优先级将被重置为集群最大优先级。**yarn-site.xml**是集群最大优先级的配置文件。
- **叶队列级优先级**：每个叶队列由管理员提供默认优先级。队列的默认优先级将用于提交的任何没有指定优先级的应用程序。**capacity-scheduler.xml** 是队列级优先级的配置文件。

#### 容量调度程序容器抢占

容量调度器支持**从资源使用超过其保证容量的队列中抢占容器**。**yarn-site.xml** 配置。

| Property                                          | Description                                                  |
| :------------------------------------------------ | :----------------------------------------------------------- |
| `yarn.resourcemanager.scheduler.monitor.enable`   | 启用一组影响调度器的定期监视器(在yarn.resourcemanager.scheduler.monitor.policies中指定)。默认值为false。 |
| `yarn.resourcemanager.scheduler.monitor.policies` | 与调度器交互的SchedulingEditPolicy类列表。配置的策略需要与调度器兼容。默认值为`org.apache.hadoop.yarn.server.resourcemanager.monitor.capacity.ProportionalCapacityPreemptionPolicy` |

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.resourcemanager.monitor.capacity.preemption.observe_only` | 如果为真，运行策略，但不影响使用抢占和杀死事件的集群。默认值为false |
| `yarn.resourcemanager.monitor.capacity.preemption.monitoring_interval` | 调用该比例容量抢占策略之间的时间(毫秒)。默认值是3000         |
| `yarn.resourcemanager.monitor.capacity.preemption.max_wait_before_kill` | 从应用程序请求抢占到终止容器之间的时间(毫秒)。默认值是15000  |
| `yarn.resourcemanager.monitor.capacity.preemption.total_preemption_per_round` | 单个回合中被抢占的资源的最大百分比。通过控制这个值，可以控制容器从集群中回收的速度。在计算所需的总优先购买量之后，策略将其伸缩回这个限制内。默认值为0.1 |
| `yarn.resourcemanager.monitor.capacity.preemption.max_ignored_over_capacity` | 抢占时超过目标容量的最大资源会被忽略。这在目标容量周围定义了一个死区，它有助于防止在计算的目标平衡周围发生颠簸和振荡。较高的值会减慢到容量的时间，并且它可能会阻止收敛到保证的容量。默认值为0.1 |
| `yarn.resourcemanager.monitor.capacity.preemption.natural_termination_factor` | 给定一个计算的抢占目标，考虑容器自然过期并只抢占增量的这个百分比。这决定了几何收敛到死区(MAX_IGNORED_OVER_CAPACITY)的速度。例如，值为0.5时，在5 * #WAIT_TIME_BEFORE_KILL范围内，即使没有自然终止，也会回收将近95%的资源。默认值是0.2 |

**capacity-scheduler.xml**配置

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.<queue-path>.disable_preemption`    | 可以将此配置设置为true，以**禁用提交到给定队列的应用程序容器的抢占**。如果未为队列设置此属性，则该属性值将从队列的父队列继承。默认值为false。 |
| `yarn.scheduler.capacity.<queue-path>.intra-queue-preemption.disable_preemption` | 此配置设置为true，以**禁用提交到给定队列的应用程序容器的队列内抢占**。如果未为队列设置此属性，则该属性值将从队列的父队列继承。默认值为false。 |

- 上面两个配置都需要 `monitor.enable`为true，且`monitor.policies`为`ProportionalCapacityPreemptionPolicy`才会起作用；
- `intra-queue-preemption.disable_preemption` 还需要 `yarn.resourcemanager.monitor.capacity.preemption.intra-queue-preemption.enabled` 为 true时，才会起作用。

#### 预订属性

控制预订的创建、删除、更新和列表。注意，任何用户都可以更新、删除或列出他们自己的预订。如果启用了预订acl但没有定义，那么每个人都可以访问。

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.root.<queue>.acl_administer_reservations` | ACL**控制谁可以*管理*给定队列的预订**。如果给定用户/组在给定队列上有必要的acl，他们可以提交、删除、更新和列出所有预订。如果未指定，此属性*的acl不会*从父队列继承。 |
| `yarn.scheduler.capacity.root.<queue>.acl_list_reservations` | ACL用于**控制谁可以`list`预订到给定队列**。如果给定用户/组在给定队列上有必要的acl，则可以列出所有应用程序。如果未指定，则不会从父队列继承此属性的acl。 |
| `yarn.scheduler.capacity.root.<queue>.acl_submit_reservations` | ACL，用**于控制谁可以向给定队列提交预订**。如果给定用户/组在给定队列上有必要的acl，则可以提交预订。如果未指定，则不会从父队列继承此属性的acl。 |

#### 容量调度器配置预留系统

容量调度器支持预留系统，**允许用户提前预留资源**。应用程序可以在运行时通过在提交期间指定reservationId来请求保留的资源。可以在**yarn-site.xml**中配置ReservationSystem的参数。

| roperty                                                      | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.resourcemanager.reservation-system.enable`             | 默认false，不启用                                            |
| `yarn.resourcemanager.reservation-system.class`              | `ReservationSystem`的类名，`CapacityScheduler`调度器则是`CapacityReservationSystem` |
| `yarn.resourcemanager.reservation-system.plan.follower`      | 在定时器上运行的PlanFollower的类名，它将`Plan`和容量调度器同步。`CapacityScheduler`调度器则是`CapacitySchedulerPlanFollower` |
| `yarn.resourcemanager.reservation-system.planfollower.time-step` | 默认1000，`PlanFollower`定时的频率，毫秒                     |

ReservationSystem可以为任何Leaf Queue配置，支持以下参数

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.<queue-path>.reservable`            | 默认值为false，即默认情况下LeafQueues中不启用预订。          |
| `yarn.scheduler.capacity.<queue-path>.reservation-agent`     | ReservationAgent的实现的类名，尝试将用户的预订请求放入`Plan`中，默认*org.apache.hadoop.yarn.server.resourcemanager.reservation.planning.AlignedPlannerWithGreedy* |
| `yarn.scheduler.capacity.<queue-path>.reservation-move-on-expiry` | 当相关的预约到期时，是否应该将应用程序移动或终止到父可预约队列。默认值为true，表示应用程序将移动到可保留队列。 |
| `yarn.scheduler.capacity.<queue-path>.show-reservations-as-queues` | 默认值为false，即在调度器UI中显示或隐藏预订队列              |
| `yarn.scheduler.capacity.<queue-path>.reservation-policy`    | 确定SharingPolicy实现的类名，如果新的保留没有违反任何不变量，该策略将验证。默认*org.apache.hadoop.yarn.server.resourcemanager.reservation.CapacityOverTimePolicy* |
| `yarn.scheduler.capacity.<queue-path>.reservation-window`    | 默认值是一天。表示如果满足计划中的约束，共享策略将验证的毫秒时间。 |
| `yarn.scheduler.capacity.<queue-path>.instantaneous-max-capacity` | 以百分比(%)表示共享策略允许单个用户保留的最大容量。默认值为1，即100%。 |
| `yarn.scheduler.capacity.<queue-path>.average-capacity`      | 在ReservationWindow上以百分比(%)聚合的允许单个用户保留的平均容量。默认值为1，即100%。 |
| `yarn.scheduler.capacity.<queue-path>.reservation-planner`   | 确定计划器实现的类名，如果计划容量低于用户保留资源(由于计划维护或节点故障)，将调用该实现。*org.apache.hadoop.yarn.server.resourcemanager.reservation.planning.SimpleCapacityReplanner*扫描计划，并以逆向接受顺序(LIFO)贪婪地删除保留/预定，直到保留的资源在计划的能力之内 |
| `yarn.scheduler.capacity.<queue-path>.reservation-enforcement-window` | 表示如果满足计划中的约束，计划者将验证的毫秒时间。默认值是1小时。 |

#### 动态创建和管理LeafQueue

CapacityScheduler支持**在父队列下自动创建叶队列**，父队列已配置为支持此特性。

- 通过队列映射设置动态自动创建的叶队列

`yarn.scheduler.capacity.queue-mappings`中的**user-group queue mapping(s) **需要指定一个额外的父队列参数，以标识需要在哪个父队列下创建自动创建的叶队列。

示例：注意`u:%user:parent1.%user`这种写法

```xml
 <property>
   <name>yarn.scheduler.capacity.queue-mappings</name>
   <value>u:user1:queue1,u:user2:%primary_group,u:%user:parent1.%user</value>
   <description>
     Here, u:%user:parent1.%user mapping allows any <user> other than user1,
     user2 to be mapped to its own user specific leaf queue which
     will be auto-created under <parent1>.
   </description>
 </property>
```

- 用于动态叶队列自动创建和管理的父队列配置

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.<queue-path>.auto-create-child-queue.enabled` | **为指定的父队列启用自动叶队列创建，默认false**，即夫队列不会自动创建子队列。 |
| `yarn.scheduler.capacity.<queue-path>.auto-create-child-queue.management-policy` | 确定“AutoCreatedQueueManagementPolicy”的实现，将**动态管理父队列下的叶队列及其容量**。默认是*org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.queuemanagement.GuaranteedOrZeroCapacityOverTimePolicy*，用户或组可能在有限的时间内向自动创建的叶队列提交应用程序并停止使用它们。因此，在父队列下自动创建的叶队列数量可能超过其保证的容量。*当前策略实现根据父队列上的容量可用性和跨叶队列的应用程序提交顺序按**最佳努力**分配已配置或零容量。* |

- 使用CapacityScheduler配置自动创建的叶队列

**支持为自动创建叶队列的自动配置配置模板参数**。自动创建的队列**支持除队列ACL、绝对资源配置之外的所有叶队列配置参数**。当前从父队列i继承队列acl。

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.<queue-path>.leaf-queue-template.capacity` | 指定自动创建的叶队列的最小保证容量。目前，自动创建的叶队列不支持绝对资源配置 |
| `yarn.scheduler.capacity.<queue-path>.leaf-queue-template.<leaf-queue-property>` | 可以在自动创建的叶队列上配置的其他队列参数，如最大容量、用户限制因子、最大资源百分比等。 |

示例：

```xml
<property>
   <name>yarn.scheduler.capacity.root.parent1.auto-create-child-queue.enabled</name>
   <value>true</value>
 </property>
 <property>
    <name>yarn.scheduler.capacity.root.parent1.leaf-queue-template.capacity</name>
    <value>5</value>
 </property>
 <property>
    <name>yarn.scheduler.capacity.root.parent1.leaf-queue-template.maximum-capacity</name>
    <value>100</value>
 </property>
 <property>
    <name>yarn.scheduler.capacity.root.parent1.leaf-queue-template.user-limit-factor</name>
    <value>3.0</value>
 </property>
 <property>
    <name>yarn.scheduler.capacity.root.parent1.leaf-queue-template.ordering-policy</name>
    <value>fair</value>
 </property>
 <property>
    <name>yarn.scheduler.capacity.root.parent1.GPU.capacity</name>
    <value>50</value>
 </property>
 <property>
     <name>yarn.scheduler.capacity.root.parent1.accessible-node-labels</name>
     <value>GPU,SSD</value>
   </property>
 <property>
     <name>yarn.scheduler.capacity.root.parent1.leaf-queue-template.accessible-node-labels</name>
     <value>GPU</value>
  </property>
 <property>
    <name>yarn.scheduler.capacity.root.parent1.leaf-queue-template.accessible-node-labels.GPU.capacity</name>
    <value>5</value>
 </property>
```

- 为自动创建的队列管理调度编辑策略配置

管理员需要在`yarn.resourcemanager.scheduler.monitor.policies`中以逗号分隔的字符串形式向当前调度编辑策略列表指定额外的`org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.QueueManagementDynamicEditPolicy`调度编辑策略。

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.resourcemanager.monitor.capacity.queue-management.monitoring-interval` | `QueueManagementDynamicEditPolicy`策略调用之间的时间(毫秒)。默认值是1500 |

#### 其它属性

- 资源计算器

| Property                                      | Description                                                  |
| :-------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.resource-calculator` | 默认是`org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator` ，仅使用内存。`DominantResourceCalculator` 使用支配资源比较多维资源，如内存，CPU等。 |

- 数据本地性

| Property                                                 | Description                                                  |
| :------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.node-locality-delay`            | 容量调度器尝试调度机架本地容器而错过的调度机会的数量。通常，这应该设置为集群中的节点数。默认-1，禁用数据本地性特性。 |
| `yarn.scheduler.capacity.rack-locality-additional-delay` | 3.x版本。<br />超过`node-locality-delay`的额外错过调度机会的数量，在此之后，容量调度器尝试调度off-switch容器。默认-1，此时计算公式为L*C/N，L是节点数或机架数，C是请求容器数，N是集群的规模。 |

Note：**将YARN与文件系统分开部署，应该禁用该特性**。

- 每个NodeManager心跳的容器分配

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.capacity.per-node-heartbeat.multiple-assignments-enabled` | 是否允许在一个NodeManager心跳中分配多个容器。默认值为true。  |
| `yarn.scheduler.capacity.per-node-heartbeat.maximum-container-assignments` | 如果`multiple-assignments-enabled`为true，表示在一个NodeManager心跳中可以分配的容器的最大数量。默认值为100，它将每个心跳分配的容器的最大数量限制为100。将此值设置为-1将禁用此限制。 |
| `yarn.scheduler.capacity.per-node-heartbeat.maximum-offswitch-assignments` | 如果`multiple-assignments-enabled`为true，表示在一个NodeManager心跳中可以分配的off-switch容器的最大数量。默认值为1，表示在一个心跳中只允许一个off-switch分配。 |



### 队列配置变更

更改队列/调度器属性和添加/删除队列可以通过两种方式完成，通过文件或通过API。

`yarn.scheduler.configuration.store.class`

- `file`：通过文件，默认。
- `memory`：通过API，但是重启时不会持久化改动。
- `leveldb`：通过API，并持久化改动到leveldb。
- `zk`：通过API，并持久化改动到zookeeper。

#### 通过文件变更

**手动**更改**conf/capacity-scheduler.xml**，并运行 **`yarn rmadmin -refreshQueues`**。

如果通过文件删除队列，需要

1）先停止队列：队列状态需要改变为`STOPPED`，删除父队列前，所有其子队列必须为`STOPPED`；

2）再删除队列：将队列配置从文件中移除；

#### 通过API变更

Note：alpha版本，后续可能会变更。**yarn-site.xml** 中配置以下参数

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.configuration.store.class`                   | backing store的类型                                          |
| `yarn.scheduler.configuration.mutation.acl-policy.class`     | 配置ACL策略来限制哪些用户可以修改哪些队列。默认是*org.apache.hadoop.yarn.server.resourcemanager.scheduler.DefaultConfigurationMutationACLPolicy*，只允许yarn管理员变更配置。可选*org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.conf.QueueAdminConfigurationMutationACLPolicy*，允许队列管理员变更该队列配置。 |
| `yarn.scheduler.configuration.store.max-logs`                | 如果使用leveldb或zookeeper，配置更改会在备份存储中被审核记录。此配置控制要存储的审计日志的最大数量，超过最大数量时将删除最老的日志。默认是1000。 |
| `yarn.scheduler.configuration.leveldb-store.path`            | 使用leveldb时配置的存储路径。默认值是${hadoop.tmp.dir}/yarn/system/confstore。 |
| `yarn.scheduler.configuration.leveldb-store.compaction-interval-secs` | 当使用leveldb时，压缩配置的时间间隔以秒为单位存储。默认值是86400，或者一天。 |
| `yarn.scheduler.configuration.zk-store.parent-path`          | 使用zookeeper时，配置的zookeeper根节点路径存储相关信息。默认值为/confstore。 |

通过 `yarn schedulerconf`命令行进行队列配置修改，或通过`http://rm-http-address:port/ws/v1/cluster/scheduler-conf`的REST API。

### Container更新(Experimental)

一旦AM从RM接收到容器，它可以请求RM更新容器的某些属性。

目前只支持两种类型的容器更新：

- **资源更新**：AM可以请求RM**更新容器的资源大小**。例如将容器从2GB 2vcore容器更改为4GB 2vcore容器；
- **执行类型更新**：AM可以请求RM更新容器的执行类型。如将执行类型从*GUARANTEED* 改为*OPPORTUNISTIC* 。

更新类型有四种：

```protobuf
enum ContainerUpdateTypeProto {
  INCREASE_RESOURCE = 0;
  DECREASE_RESOURCE = 1;
  PROMOTE_EXECUTION_TYPE = 2;
  DEMOTE_EXECUTION_TYPE = 3;
}
```

**DECREASE_RESOURCE** 和**DEMOTE_EXECUTION_TYPE** 的容器**更新是自动**的，AM不必显式地要求NM减少容器的资源，其他更新类型要求AM显式地请求NM更新容器。

如果`yarn.resourcemanager.auto-update.containers`参数设置为true(默认为false)， RM将确保所有容器更新都是自动的，所有container更新会在下个心跳时发送给NM。



## 公平调度

与默认的Hadoop调度器形成一个应用程序队列不同，它允许短应用程序在合理的时间内完成，而不会饿死长应用程序。这也是在多个用户之间共享集群的一种合理方式。最后，公平分享也与应用程序的优先级有关—**优先级被用作权重来确定每个应用程序应该获得的总资源的比例**。

**调度器将应用程序进一步组织到“队列”中，并在这些队列之间公平地共享资源**。

- **默认情况下，所有用户共享一个名为“default”的队列**；
- 如果应用程序在容器资源请求中特别列出一个队列，则该请求被提交到该队列。还可以通过配置根据请求中包含的用户名分配队列。
- 在每个队列中，使用调度策略在运行的应用程序之间共享资源。默认是基于内存的公平共享，但是也可以配置FIFO和具有优势资源公平性的多资源。
- 队列可以安排在层次结构中以划分资源，并配置权重以按特定比例共享集群。



### 分层队列和插件化策略

`root`是默认队列，可用资源按照典型的公平调度方式分布在根队列的子队列中，子队列同样将资源公平分配给其子队列。应用程序只能在叶队列上调度。

**队列名称以其父队列的名称开始**，`.`作为分隔符。因此根队列下名为“queue1”的队列将被称为“root”。而在名为parent1的队列下的名为queue2的队列称为root.parent1.queue2。

**在引用队列时，名称的根部分是可选的**，因此可以仅将queue1称为“queue1”，将queue2称为“parent1.queue2”。

Fair scheduler允许**为每个队列设置不同的自定义策略**，以允许以用户希望的任何方式共享队列的资源。

通过扩展`org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.SchedulingPolicy`可以实现自定义策略。内置策略有 `FifoPolicy`，`FairSharePolicy` (default) 和 `DominantResourceFairnessPolicy `。



### 自动将应用放到队列

Fair Scheduler允许管理员**配置自动将提交的应用程序放入适当队列的策略**。位置取决于提交者的用户和组以及应用程序传递的请求队列。**策略由一组规则组成**，按顺序应用这些规则对传入应用程序进行分类。**每个规则要么将应用程序放入队列，要么拒绝它，要么继续执行下一个规则**。



### 配置

Fair-Schedule的调度涉及到两个文件：

- yarn-site.xml文件中添加配置属性来设置调度器范围的选项；
- 分配文件，列出存在哪些队列以及它们各自的权重和容量。

**分配文件每10秒重新加载一次，允许动态地进行更改**。

#### yarn-site.xml

```xml
<property>
  <name>yarn.resourcemanager.scheduler.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>
```

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.scheduler.fair.allocation.file`                        | 分配文件的路径。如果给出了相对路径，则在类路径(通常包括Hadoop conf目录)上搜索该文件。默认为**fair-scheduler.xml**。 |
| `yarn.scheduler.fair.user-as-default-queue`                  | 默认值为true。在没有指定队列名称的情况下，是否使用与分配关联的用户名作为默认队列名称。如果设置**为“false”或未设置，则所有作业都有一个共享的默认队列，名为“default”**。如果在分配文件中给出了队列放置策略，则忽略此属性。 |
| `yarn.scheduler.fair.preemption`                             | 是否使用抢占。默认值为false。                                |
| `yarn.scheduler.fair.preemption.cluster-utilization-threshold` | 当利用率高于此阈值时，尝试抢占。利用率计算为所有资源的最大使用率与容量比率。默认为0.8f。 |
| `yarn.scheduler.fair.sizebasedweight`                        | 是否根据每个应用程序的大小分配份额，而不是不分大小给所有应用程序相同的份额。当设置为true时，应用程序的加权值为1的自然对数加上应用程序的总请求内存，再除以2的自然对数。默认值为false。 |
| `yarn.scheduler.fair.assignmultiple`                         | 是否允许在一个心跳中分配多个容器。默认值为false。            |
| `yarn.scheduler.fair.dynamic.max.assign`                     | 如果assignmultiple为true，是否动态地确定在一个心跳中可以分配的资源数量。当打开时，节点上大约有一半的未分配资源在单个心跳中分配给容器。默认值为true。 |
| `yarn.scheduler.fair.max.assign`                             | 如果assignmultiple为true且dynamic.max.assign为false，表示一次心跳可以分配的容器的最大数量。0到1之间的浮点数，默认值为-1，没有限制。 |
| `yarn.scheduler.fair.locality.threshold.node`                | 对于请求特定节点上的容器的应用程序，自最后一次容器分配后在接受另一个节点上的放置之前等待的调度机会的数量。默认值-1.0表示不要错过任何调度机会。 |
| `yarn.scheduler.fair.locality.threshold.rack`                | 对于请求特定机架上的容器的应用程序，自最后一次容器分配后在接受另一个机架上的放置之前等待的调度机会的数量。表示为0到1之间的浮点数，默认值-1.0表示不要错过任何调度机会。 |
| `yarn.scheduler.fair.allow-undeclared-pools`                 | 如果true，可以在应用程序提交时创建新的队列；false时，如果应用程序没有在分配文件中指定队列，则在“default"队列。默认值为true。如果在分配文件中给出了队列放置策略，则忽略此属性。 |
| `yarn.scheduler.fair.update-interval-ms`                     | 默认值为500毫秒。锁定调度器和重新计算公平份额，重计算需求和检查是否需要先抢占的间隔。 |
| `yarn.resource-types.memory-mb.increment-allocation`         | fairscheduler按此值的增量授予内存。如果您提交的任务的资源请求不是其倍数，则请求将四舍五入到最接近的增量。默认值为1024 MB。 |
| `yarn.resource-types.vcores.increment-allocation`            | fairscheduler按此值的增量授予vcores。如果您提交的任务的资源请求不是其倍数。增量分配时，请求将四舍五入到最接近的增量。默认为1。 |
| `yarn.resource-types.<resource>.increment-allocation`        | fairscheduler按此值的增量授予<resource>。如果您提交的任务的资源请求不是其倍数。增量分配时，请求将四舍五入到最接近的增量。 |
| `yarn.scheduler.increment-allocation-mb`                     | 使用yarn.resource-types.memory-mb.increment-allocation替代   |
| `yarn.scheduler.increment-allocation-vcores`                 | 使用yarn.resource-types.vcores.increment-allocation替代      |

#### 分配文件的格式

XML格式，主要包含5种元素类型

- **Queue elements**：表示队列，属性`type`可以设置为为`parent`作为父队列。每个队列包含以下配置：
  - **minResources**：队列有权获得的**最小资源**。如果一个队列的最小共享没有得到满足，那么它将在同一父队列下的任何其他队列之前被提供可用资源。
  - **maxResources**：队列可以分配的**最大资源**。不会为队列分配一个容器，使其总使用量超过此限制。此限制是递归强制执行的，如果该分配将使队列或其父队列超过最大资源，则不会给队列分配容器。
  - **maxContainerAllocation**：队列可以分配给**单个容器的最大资源**，如果未设置该属性，则其值从父队列继承。默认时**maximum-allocation-mb**和**maximum-allocation-vcores**。此属性对根队列无效。
  - **maxChildResources**：**特定子队列**可以分配的最大资源。
    - 前四个的配置格式：“vcores=X, memory-mb=Y”（单位从资源的默认配置单元推断） 或者 “vcores=X%, memory-mb=Y%”。
  - **maxRunningApps**：限制队列中要**同时运行的应用程序的数量**
  - **maxAMShare**：限制**在队列用于运行AM的的公平份额**。**此属性只能用于叶队列。**如果设置为1.0f，那么叶队列中的AMs可以占用100%的内存和CPU公平份额。-1.0f将禁用此功能，amShare将不会被检查。默认值为0.5f。
  - **weight**：**与其他队列非比例地共享集群**，默认1.
  - **schedulingPolicy**：设置任何**队列的调度策略**，默认为fair
  - **aclSubmitApps**：可以向**队列提交应用程序的用户和/或组列表**。
  - **aclAdministerApps**：可以**管理队列的用户和/或组的列表**。目前唯一的管理行为是终止应用程序。
  - **minSharePreemptionTimeout**：在**尝试抢占容器以从其他队列获取资源之前，队列低于其最小共享的秒数**。如果未设置，则队列将从其父队列继承该值。默认值是Long.MAX_VALUE，这意味着在您设置有意义的值之前，它不会抢占容器。
  - **fairSharePreemptionTimeout**：在**尝试抢占容器以从其他队列获取资源之前，队列低于其公平共享阈值的秒数**。如果未设置，则队列将从其父队列继承该值。默认值是Long.MAX_VALUE，这意味着在您设置有意义的值之前，它不会抢占容器。
  - **fairSharePreemptionThreshold**：**队列的公平共享抢占阈值**，如果队列等待fairSharePreemptionTimeout而没有接收fairSharePreemptionThreshold*fairShare资源，则允许它抢占容器以从其他队列获取资源。如果未设置，则队列将从其父队列继承该值。默认值为0.5f。
  - **allowPreemptionFrom**：确定**是否允许调度程序抢占队列中的资源**。默认值为true。如果队列将此属性设置为false，则此属性将递归应用于所有子队列。
  - **reservation**：向**ReservationSystem表示队列的资源可供用户预留**。这只适用于叶队列。如果没有配置此属性，则不能保留叶队列。
- **User elements**：管理单个用户行为的设置，包含单个属性`maxRunningApps`，对特定用户运行的应用程序数量的限制。
- **A userMaxAppsDefault element**：没有指定限制的用户设置默认的运行应用程序限制。
- **A defaultFairSharePreemptionTimeout element**
- **A defaultMinSharePreemptionTimeout element**
- **A defaultFairSharePreemptionThreshold element**
- **A queueMaxAppsDefault element**
- **A queueMaxResourcesDefault element**
- **A queueMaxAMShareDefault element**
- **A defaultQueueSchedulingPolicy element**
- **A reservation-agent element**：ReservationAgent实现的类名，试图将用户的预订请求放入`Plan`中，默认org.apache.hadoop.yarn.server.resourcemanager.reservation.planning.AlignedPlannerWithGreedy
- **A reservation-policy element**：SharingPolicy实现的类名，验证新的保留是否没有违反任何不变量，默认org.apache.hadoop.yarn.server.resourcemanager.reservation.CapacityOverTimePolicy
- **A reservation-planner element**：Planner实现的类名，如果计划容量低于用户保留资源(由于计划维护或节点故障)，则调用该类。默认org.apache.hadoop.yarn.server.resourcemanager.reservation.planning.SimpleCapacityReplanne，扫描计划，并以后进先出(LIFO)的顺序贪婪地删除保留，直到保留的资源在计划容量之内。
- **A queuePlacementPolicy element**：包含规则元素列表，告诉调度器如何将传入的应用程序放入队列中。所有规则都接受**“create”参数（默认为true），该参数指示规则是否可以创建一个新队列**。如果设置为false，并且该规则将把应用程序放在分配文件中未配置的队列中，将继续执行下一个规则。最后一个规则必须是不能发出continue的规则。有效的规则：
  - **specified**：**应用程序被放置到它请求的队列中**。如果应用程序没有请求队列，即指定“default”。如果应用程序请求一个以句点开头或结尾的队列名称，例如".q1"或"q1."将被拒绝；
  - **user**：应用程序被放入一个带有提交它的用户名称的队列中，用户名中的`.`会被替换成`__dot__`；
  - **primaryGroup**：应用程序被放置到，具有提交应用程序的用户的主组的队列，组名中的`.`会被替换成`__dot__`；
  - **secondaryGroupExistingQueue**：应用程序被放入与提交它的**用户的 secondary组**相匹配的队列，选择与已配置队列匹配的第一个secondary组，组名中的`.`会被替换成`__dot__`；
  - **nestedUserQueue**：应用程序被放置到一个具有用户名的队列中，该队列位于嵌套规则建议的队列之下。' nestedUserQueue '规则，用户队列可以在任何父队列下创建，而' user '规则只在根队列下创建用户队列。**注意，只有当嵌套规则返回父队列时，才会应用nestedUserQueue规则。**
  - **default**：应用程序被放置在**其“queue”属性中指定的队列**中。如果没有指定' queue '属性，应用程序将被放置到' root.default '队列中。
  - **reject**：拒绝这个app

示例

```xml
<?xml version="1.0"?>
<allocations>
  <queue name="sample_queue">
    <minResources>10000 mb,1vcores</minResources>
    <maxResources>vcores=2000, memory-mb=2</maxResources>
    <maxRunningApps>50</maxRunningApps>
    <maxAMShare>0.1</maxAMShare>
    <weight>2.0</weight>
    <schedulingPolicy>fair</schedulingPolicy>
    <queue name="sample_sub_queue">
      <aclSubmitApps>charlie</aclSubmitApps>
      <minResources>5000 mb,0vcores</minResources>
    </queue>
    <queue name="sample_reservable_queue">
      <reservation></reservation>
    </queue>
  </queue>

  <queueMaxAMShareDefault>0.5</queueMaxAMShareDefault>
  <queueMaxResourcesDefault>40000 mb,0vcores</queueMaxResourcesDefault>

  <!-- Queue 'secondary_group_queue' is a parent queue and may have
       user queues under it -->
  <queue name="secondary_group_queue" type="parent">
  <weight>3.0</weight>
  <maxChildResources>4096 mb,4vcores</maxChildResources>
  </queue>

  <user name="sample_user">
    <maxRunningApps>30</maxRunningApps>
  </user>
  <userMaxAppsDefault>5</userMaxAppsDefault>

  <queuePlacementPolicy>
    <rule name="specified" />
    <rule name="primaryGroup" create="false" />
    <rule name="nestedUserQueue">
        <rule name="secondaryGroupExistingQueue" create="false" />
    </rule>
    <rule name="default" queue="sample_queue"/>
  </queuePlacementPolicy>
</allocations>
```

### ACL

队列访问控制列表(acl)允许管理员控制谁可以对特定队列采取操作。

它们配置了`aclSubmitApps`和`aclAdministerApps`属性，可以对每个队列进行设置。目前唯一支持的管理操作是终止应用程序。

属性的值格式类似于`user1,user2 group1,group2`或` group1,group2`。

具有父队列的ACL权限可以操作子队列。

**根队列的acl默认为“\*”**，因为acl是向下传递的，这意味着每个人都可以向每个队列中的应用程序提交和终止应用程序。要开始限制访问，请将根队列的acl更改为"\*"以外的内容。

#### 预约ACL

预约访问控制列表允许管理员控制谁可以对特定队列采取预约操作。

配置了`acladministerreservation`、`aclListReservations`和`aclsubmitreservation`属性，可以为每个队列设置这些属性。目前支持的管理操作是更新和删除预订。管理员还可以提交并在队列中列出所有预订。

属性的值格式类似于`user1,user2 group1,group2`或` group1,group2`。

如果用户/组是预约ACL的成员，则允许对队列进行操作。注意，任何用户都可以更新、删除或列出他们自己的预订。如果启用了预订acl但没有定义，那么每个人都可以访问。

#### 配置 `ReservationSystem`

Fair Scheduler支持ReservationSystem，**允许用户提前预留资源**。应用程序可以在运行时通过在提交期间指定reservationId来请求保留的资源。可以在ReservationSystem的yarn-site.xml中配置以下配置参数。

| Property                                                     | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `yarn.resourcemanager.reservation-system.enable`             | 在ResourceManager中启用ReservationSystem，默认false          |
| `yarn.resourcemanager.reservation-system.class`              | ReservationSystem 的类名，FairScheduler的默认是FairReservationSystem |
| `yarn.resourcemanager.reservation-system.plan.follower`      | 在定时器上运行并和ResourceScheduler和Plan同步。FairScheduler默认FairSchedulerPlanFollower |
| `yarn.resourcemanager.reservation-system.planfollower.time-step` | PlanFollower定时器频率(毫秒)的。默认1000。                   |

ReservationSystem集成了Fair Scheduler队列层次结构，可以为叶队列(且仅为叶队列)配置。



### 管理（Administration）

fair scheduler通过一些机制提供对运行时管理的支持:

#### 运行时更改配置

通过编辑分配文件，可以在运行时**修改最小共享、限制、权重、抢占超时和队列调度**策略。**调度器将在看到该文件被修改后10-15秒重新加载该文件。**

#### 通过web UI进行监控

http://\*ResourceManager URL\*/cluster/scheduler 可以查看当前应用，队列和共享份额。

在web界面上，可以看到每个队列的以下字段：

- **Used Resources** - 分配给队列内容器的资源总和。
- **Num Active Applications** - 队列中已接收至少一个容器的应用程序的数量。
- **Num Pending Applications** - 队列中尚未接收到任何容器的应用程序的数量。
- **Min Resources** - 保证队列配置的最小资源。
- **Max Resources** - 已配置的队列允许使用的最大资源。
- **Instantaneous Fair Share** - 队列资源的瞬时公平共享。这些共享只考虑活动队列(那些运行应用程序的队列)，并用于调度决策。当其他队列不使用份额时，可以为队列分配超出其共享的资源。如果队列的资源消耗处于或低于它的瞬时公平共享，那么它的容器永远不会被抢占。
- **Steady Fair Share** -队列中资源的稳定公平的份额。这些共享会考虑所有队列，而不管它们是否处于活动状态(是否有正在运行的应用程序)。这些计算频率较低，只有在配置或容量更改时才会更改。它们旨在为用户提供对预期资源的可见性，并因此显示在Web UI中。

#### 队列中移动应用

Fair Scheduler支持将正在运行的应用程序移动到另一个队列。这对于将重要的应用程序移动到高优先级队列或将不重要的应用程序移动到低优先级队列非常有用。

```shell
 yarn application -movetoqueue appID -queue targetQueueName
```

当应用程序移动到队列时，**它的现有分配将与新队列的分配一起计算**，而不是与旧队列一起计算，以确定公平性。如果向队列中添加应用程序的资源会违反其maxRunningApps或maxResources约束，那么将应用程序移动到队列的尝试将会失败。

#### 转储公平调度程序状态

公平调度程序能够定期转储其状态。默认情况下是禁用的。可以将`org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler.statedump`的日志界别改为`DEBUG`。

在默认情况下，公平调度程序的日志会存储到RM的日志文件中。公平调度程序状态转储可能会生成大量日志数据。在log4j.properties中取消注释"Fair scheduler state dump"区域会将日志写到不同的文件。



## 实现

 Capacity Scheduler，它根据利用率对队列进行排序，并首先将 containers 分配给利用率最低的队列。集群的调度速度：

- 假设有两个队列 A 和 B， A 的利用率为 10%，B 的利用率为 20%，调度器将**首先为队列 A 调度 containers**，然后再移动到 B 队列，并为提供其服务；
- 在容器流失率高的环境中可能会出现短暂的死锁，在队列利用率收敛并且队列 A 的利用率超过队列 B 之前，调度程序不会接收提交到队列 B 的工作负载；
- **Linkin的PR有没有被合并？**



默认情况下，**YARN 资源管理器使用同步调度**：

- 节点心跳到资源管理器，这会**触发调度器在该节点上调度未完成的 container**；
- 如果 container 的分区和节点的分区不匹配，则不会在该节点上分配 container；
- **Linkin的PR有没有被合并？**



## 全局调度（3.3版本未实现）

> https://issues.apache.org/jira/browse/YARN-5139

现有的调度模式是基于NM（NodeManager）的心跳的方式进行调度的（不考虑持续调度模式），进行调度时，一次只考虑一个节点，这很容易造成非最优的调度。举个例子，一个请求要到第100个节点的心跳过来才是他最合适的节点，那么很可能到中间某一次心跳就分给他资源了，不是他最满意的节点。

```java
// 首先收到NM的心跳，如有空闲Container则进行调度，调度的层次是从父队列到子队列，再到对应的app，最终把应该调度的资源请求在该心跳的节点上进行资源分配。
for node in allNodes:
  	Go to parentQueue
    Go to leafQueue
      for application in leafQueue.applications:
        for resource-request in application.resource-requests
          try to schedule on node
```

**全局调度的调度逻辑为（提出候选节点的概念，不再是只考虑心跳的节点）：**

```java
Go to parentQueue
  Go to leafQueue
    for application in leafQueue.applications:
      for resource-request in application.resource-requests
        for node in nodes (node candidates sorted according to resource-request)
          try to schedule on node
```