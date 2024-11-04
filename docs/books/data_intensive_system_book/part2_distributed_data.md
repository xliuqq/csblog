# 第二部分：分布式数据

> 如果 **多台机器** 参与数据的存储和检索，会发生什么？

出于各种各样的原因，希望将数据库分布到多台机器上

- 可伸缩性：负载分散到多台计算机；
- 容错 / 高可用性：单台机器（或多台机器，网络或整个数据中心）出现故障的情况下仍然能继续工作
- 延迟：全球范围部署多个服务器，从地理上最近的数据中心获取服务。



## 伸缩到更高负载

最简单的方法就是购买更强大的机器（有时称为 **垂直伸缩**，即 vertical scaling，或 **向上伸缩**，即 scale up），即 **共享内存架构（shared-memory architecture）**。

- 问题：**成本的增长速度快于负载的线性增长**；提供**有限的容错能力**

另一种是**共享磁盘架构（shared-disk architecture）**“：多台具有独立处理器和内存的机器

- 数据存储在机器之间共享的磁盘阵列上，这些磁盘通过快速网络连接
- 竞争和锁定的开销限制了共享磁盘方法的可伸缩性$^{[1]}$

### 无共享架构

**无共享架构$^{[2]}$**（shared-nothing architecture，有时被称为 **水平伸缩**，即 horizontal scaling，或 **向外伸缩**，即 scaling out）

- 运行数据库软件的每台机器 / 虚拟机都称为 **节点（node）**；
- 每个节点只使用各自的处理器，内存和磁盘，节点间使用网络进行通信和协调；

虽然分布式无共享架构有许多优点，但它通常也会给应用带来额外的复杂度，有时也会限制你可用数据模型的表达力。

- 某些情况下，一个简单的单线程程序可以比一个拥有超过 100 个 CPU 核的集群表现得更好$^{[3]}$

### 复制 vs 分区

数据分布在多个节点上有两种常见的方式：

- 复制（Replication）：几个不同的节点上保存数据的相同副本，提供冗余；
- 分区 (Partitioning)：将一个大型数据库拆分成较小的子集，从而不同的分区可以指派给不同的 **节点**（nodes，亦称 **分片**，即 sharding）；

复制和分区是不同的机制，但它们经常同时使用。



## 参考文献

1. Ben Stopford: “[Shared Nothing vs. Shared Disk Architectures: An Independent View](http://www.benstopford.com/2009/11/24/understanding-the-shared-nothing-architecture/),” benstopford.com, November 24, 2009.
2. Michael Stonebraker: “[The Case for Shared Nothing](http://db.cs.berkeley.edu/papers/hpts85-nothing.pdf),” IEEE Database EngineeringBulletin, volume 9, number 1, pages 4–9, March 1986.
3. Frank McSherry, Michael Isard, and Derek G. Murray: “[Scalability! But at What COST?](http://www.frankmcsherry.org/assets/COST.pdf),” at 15th USENIX Workshop on Hot Topics in Operating Systems (HotOS),May 2015.