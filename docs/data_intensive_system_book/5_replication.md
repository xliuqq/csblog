---
tags:
  - 数据密集型系统 
---

# 复制

> 

出于各种各样的原因，希望能复制数据：

- 延迟：使得数据与用户在地理上接近 ，如 CDN；
- 可用性：即使系统的一部分出现故障，系统也能继续工作；
- 读取吞吐量：伸缩可以接受读请求的机器数量；

复制的困难之处在于处理复制数据的 **变更（change）**。

- 三种流行的变更复制算法：**单领导者（single leader，单主）**，**多领导者（multi leader，多主）** 和 **无领导者（leaderless，无主）**

## 领导者与追随者

> 存在多个副本时，会不可避免的出现一个问题：如何确保所有数据都落在了所有的副本上？

**基于领导者的复制（leader-based replication）** （也称 **主动/被动（active/passive）** 复制或 **主/从（master/slave）** 复制

- 其中一个副本被指定为 **领导者（leader）**，也称为 **主库（master|primary）**，其他副本被称为 **追随者（followers）**，亦称为 **只读副本（read replicas）**、**从库（slaves）**、**备库（ secondaries）** 或 **热备（hot-standby）**；		

**写入流程**：仅向领导者写入数据；

- 当客户端要向数据库写入时，它必须将请求发送给该 **领导者**；
- 当领导者将新数据写入本地存储时，它也会将数据变更发送给所有的追随者，称之为 **复制日志（replication log）** 或 **变更流（change stream）**；
- 每个跟随者从领导者拉取日志，并相应更新其本地数据库副本，方法是**按照与领导者相同的处理顺序**来进行所有写入。

读取流程：向领导者或任一追随者进行查询。

![基于领导者的（主/从）复制](pics/fig5-1.png)

### 同步复制与异步复制

同步与异步复制的示例：从库1是同步复制，从库2是异步复制

![基于领导者的复制：一个同步从库和一个异步从库](pics/fig5-2.png)

关于真实环境的使用：

- 将所有从库都设置为同步的是不切实际，一般采用 **半同步（semi-synchronous）**$^{[1]}$，一个从库同步，其它从库异步；

- 通常情况下，基于领导者的复制都配置为完全异步；

  - 已经确认写入也不能保证是 **持久（Durable）**：未复制给从库的写入在主库失效不可恢复时将会丢失；

对于异步复制系统而言，主库故障时会丢失数据可能是一个严重的问题：

- **链式复制（chain replication）**$^{[2,3]}$：研究不丢数据但仍能提供良好性能和可用性的复制方法



### 设置新的从库



### 处理节点宕机





## 参考文献

1. Yoshinori Matsunobu: “[Semi-Synchronous Replication at Facebook](http://yoshinorimatsunobu.blogspot.co.uk/2014/04/semi-synchronous-replication-at-facebook.html),” *yoshinorimatsunobu.blogspot.co.uk*, April 1, 2014.
1. Robbert van Renesse and Fred B. Schneider: “[Chain Replication for Supporting High Throughput and Availability](http://static.usenix.org/legacy/events/osdi04/tech/full_papers/renesse/renesse.pdf),” at *6th USENIX Symposium on Operating System Design and Implementation* (OSDI), December 2004.
3. Jeff Terrace and Michael J. Freedman: “[Object Storage on CRAQ: High-Throughput Chain Replication for Read-Mostly Workloads](https://www.usenix.org/legacy/event/usenix09/tech/full_papers/terrace/terrace.pdf),” at *USENIX Annual Technical Conference* (ATC), June 2009.