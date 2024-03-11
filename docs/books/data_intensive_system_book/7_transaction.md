# 事务

在数据系统的残酷现实中，很多事情都可能出错：

- 数据库软件、硬件可能在任意时刻发生故障（包括写操作进行到一半时）。
- 应用程序可能在任意时刻崩溃（包括一系列操作的中间）。
- 网络中断可能会意外切断数据库与应用的连接，或数据库之间的连接。
- 多个客户端可能会同时写入数据库，覆盖彼此的更改。
- 客户端可能读取到无意义的数据，因为数据只更新了一部分。
- 客户端之间的竞争条件可能导致令人惊讶的错误。

**事务（transaction）**是应用程序将多个读写操作组合成一个逻辑单元的一种方式

- 整个事务要么成功 **提交**（commit），要么失败 **中止**（abort）或 **回滚**（rollback）；
- 如果失败，应用程序可以安全地重试；**不用担心部分失败**的场景；

**并不是所有的应用都需要事务**，有时候弱化事务保证、或完全放弃事务也是有好处的（例如，为了获得更高性能或更高可用性）。

本章将研究许多出错案例，并**探索数据库用于防范这些问题**的算法。

- 深入 **并发控制** 的领域，讨论各种可能发生的竞争条件；
- 数据库如何实现 **读已提交（read committed）**，**快照隔离（snapshot isolation）** 和 **可串行化（serializability）** 等隔离级别。

本章同时适用于单机数据库与分布式数据库；在 [第八章](./8_trouble_in_distributed_system.md) 中将重点讨论仅出现在分布式系统中的特殊挑战。

## 事务的棘手概念

> 随着这种新型分布式数据库的炒作，人们普遍认为事务是可伸缩性的对立面，任何大型系统都必须放弃事务以保持良好的性能和高可用性$^{[1,2]}$。另一方面，数据库厂商有时将事务保证作为 “重要应用” 和 “有价值数据” 的基本要求。这两种观点都是 **纯粹的夸张**。

与其他技术设计选择一样，事务有其优势和局限性，需要进一步了解这些权衡。

### ACID的含义

ACID 由 Theo Härder 和 Andreas Reuter 于 1983 年提出，旨在为数据库中的容错机制建立精确的术语。

- 不同数据库的 ACID 实现并不相同，关于 **隔离性** 的含义就有许多含糊不清$^{[3]}$；

符合 ACID 标准的系统有时被称为 BASE：

- **基本可用性（Basically Available）**，**软状态（Soft State）** 和 **最终一致性（Eventual consistency）**$^{[4]}$
- 比 ACID 的定义更加模糊，似乎 BASE 的唯一合理的定义是 “不是 ACID”

#### 原子性（Atomicity）

ACID 的原子性并**不**是关于 **并发**，描述如果几个进程试图同时访问相同的数据会发生什么情况属于**隔离性**。

原子性：**能够在错误时中止事务，丢弃该事务进行的所有写入变更的能力**

- 如果事务被 **中止（abort）**，应用程序可以确定它没有改变任何东西，所以可以安全地重试

#### 一致性（Consistency）

一致性这个词被赋予太多含义：

- 在 [第五章](./5_replication.md) 中，我们讨论了副本一致性，以及异步复制系统中的最终一致性问题（请参阅 “[复制延迟问题](./5_replication.md#复制延迟问题)”）。
- [一致性哈希](./6_partitions.md#一致性哈希) 是某些系统用于重新分区的一种分区方法。
- 在 [CAP 定理](./9_concistency_consensus.md#CAP定理) 中，一致性一词用于表示 [线性一致性](./9_concistency_consensus.md#线性一致性)。
- 在 ACID 的上下文中，**一致性** 是指数据库在应用程序的特定概念中处于 “良好状态”。

ACID 一致性的概念是，**对数据的一组特定约束必须始终成立**，即 **不变式（invariants）**。

- 一致性（在 ACID 意义上）是应用程序的属性，字母 C 不属于 ACID[^1]

- 应用可能依赖数据库的原子性和隔离性来实现一致性，但这并不仅取决于数据库;

#### 隔离性（Isolation）

**并发** 问题（**竞争条件**，即 race conditions）：多个客户端同时访问相同的数据库记录。

- 假设你有两个客户端同时在数据库中增长一个计数器，由于竞态条件，实际上只增至 43。

![**两个客户之间的竞争状态同时递增计数器**](pics/fig7-1.png)

隔离性：意味着**同时执行的事务是相互隔离的**

- 传统的数据库教科书将隔离性形式化为 **可串行化（Serializability）**
  - 确保当多个事务被提交时，结果与它们串行运行（一个接一个）是一样的，尽管实际上它们可能是并发运行的$^{[6]}$
- 实践中很少会使用可串行的隔离，因为有性能损失，在 “[弱隔离级别](#弱隔离级别)” 中研究快照隔离和其他形式的隔离。

#### 持久性（Durability）

**持久性** 是一个承诺，即一旦事务成功完成，即使发生硬件故障或数据库崩溃，写入的任何数据也不会丢失。

- 单节点数据库：数据已被写入非易失性存储设备，还包括预写日志或类似的文件；
- 带复制的数据库：数据已成功复制到一些节点；

为了提供持久性保证，数据库必须等到这些写入或复制完成后，才能报告事务成功提交。

- **完美的持久性是不存在的** ：如果所有硬盘和所有备份同时被销毁

> #### 复制与持久性
>
> 在历史上，持久性意味着写入归档磁带。后来它被理解为写入磁盘或 SSD。再后来它又有了新的内涵即 “复制（replication）”。
>
> 没有哪种实现是完美的：
>
> - 写入磁盘然后机器宕机：即使数据未丢失，仍需等待机器修复或者磁盘迁移。复制系统可以保持可用性；
> - 一个相关性故障（停电，或一个特定输入导致所有节点崩溃的 Bug）可能会一次性摧毁所有副本，内存数据库仍要跟磁盘打交道；
> - 在异步复制系统中，当主库不可用时，最近的写入操作可能会丢失（参阅「[处理节点宕机](./5_replication.md#处理节点宕机)」）；
> - 当电源突然断电时，特别是固态硬盘，有证据显示有时会违反应有的保证：甚至 fsync 也不能保证正常工作$^{[7]}$。硬盘固件可能有错误，就像任何其他类型的软件一样$^{[8,9]}$。
> - 存储引擎和文件系统之间的微妙交互可能会导致难以追踪的错误，并可能导致磁盘上的文件在崩溃后被损坏$^{[10,11]}$。
> - 磁盘上的数据可能会在没有检测到的情况下逐渐损坏$^{[12]}$。需要尝试从历史备份中恢复数据。
> - 固态硬盘的研究发现，在运行的前四年中，30% 到 80% 的硬盘会产生至少一个坏块$^{[13]}$。相比固态硬盘，磁盘的坏道率较低，但完全失效的概率更高。
> - 如果 SSD 断电，可能会在几周内开始丢失数据，具体取决于温度$^{[14]}$。
>
> 在实践中，没有一种技术可以提供绝对保证。**只有各种降低风险的技术**，包括写入磁盘，复制到远程机器和备份 —— 它们可以且应该一起使用。与往常一样，最好抱着怀疑的态度接受任何理论上的 “保证”。

### 单对象和多对象操作

假设你想同时修改多个对象（行，文档，记录）。通常需要 **多对象事务（multi-object transaction）** 来保持多块数据同步。

示例：执行以下查询来显示用户未读邮件数量：

```sql
SELECT COUNT（*）FROM emails WHERE recipient_id = 2 AND unread_flag = true
```

- 单独的字段存储未读邮件的数量（一种反规范化）：新消息写入时，需要同时修改未读计数器；
- 下图是**违反隔离性的示例**：用户 2 看到邮件列表里显示有未读消息，但计数器显示为零未读消息；

![**违反隔离性：一个事务读取另一个事务的未被执行的写入（“脏读”）**](pics/fig7-2.png)

- 下面是对**原子性的要求**：如果对计数器的更新失败，事务将被中止，并且插入的电子邮件将被回滚。

![**原子性确保发生错误时，事务先前的任何写入都会被撤消，以避免状态不一致**](pics/fig7-3.png)

多对象事务需要某种方式来**确定哪些读写操作属于同一个事务**。

- 关系型数据库：基于客户端与数据库服务器的 TCP 连接
  - 在任何特定连接上，`BEGIN TRANSACTION` 和 `COMMIT` 语句之间的所有内容，被认为是同一事务的一部分[^2]

- 许多非关系数据库并没有将这些操作组合在一起的方法
  - 即使存在多对象 API（例如，某键值存储可能具有在一个操作中更新几个键的 multi-put 操作），但这并不一定意味着它具有事务语义

#### 单对象写入

当单个对象发生改变时，原子性和隔离性也是适用的：例如，写入 20KB 的 JSON 文档

- 如果在发送第一个 10 KB 之后网络连接中断，数据库是否存储了不可解析的 10KB JSON 片段？
- 如果在数据库正在覆盖磁盘上的前一个值的过程中电源发生故障，是否最终将新旧值拼接在一起？
- 如果另一个客户端在写入过程中读取该文档，是否会看到部分更新的值？

存储引擎一个几乎普遍的目标是：**对单节点上的单个对象（例如键值对）上提供原子性和隔离性**。

- 原子性：可以通过使用日志来实现崩溃恢复；
- 隔离性：使用每个对象上的锁来实现隔离（每次只允许一个线程访问对象） ；

一些数据库也提供更复杂的原子操作[^3]：如自增操作，CAS 操作

- 防止在多个客户端尝试同时写入同一个对象时丢失更新（请参阅 “[防止丢失更新](#防止丢失更新)”）；
- CAS 以及其他单一对象操作被称为 “轻量级事务”，不是通常意义上的事务；

事务通常被理解为，**将多个对象上的多个操作合并为一个执行单元的机制**。

#### 多对象事务的需求

许多分布式数据存储已经放弃了多对象事务，因为多对象事务很难跨分区实现，而且在需要高可用性或高性能的情况下。

- 但在分布式数据库中实现事务，并没有什么根本性的障碍。[第九章](./9_consistency_consensus.md) 将讨论分布式事务的实现。

许多其他场景需要协调写入几个不同的对象：需要多对象事务

- 关系数据模型中的外键引用：当插入几个相互引用的记录时，外键必须是正确的和最新的；
- 文档数据模型：缺乏连接功能的文档数据库会鼓励非规范化（请参阅 “[关系型数据库与文档数据库在今日的对比](./2_data_model_query_language.md#关系型数据库和文档数据库的对比)”）；
- 次级索引的数据库：每次更改值时都需要更新索引；

这些应用仍然可以在没有事务的情况下实现。然而，**没有原子性，错误处理就要复杂得多，缺乏隔离性，就会导致并发问题**。

- 在 “[弱隔离级别](#弱隔离级别)” 中讨论这些问题，并在 [第十二章](12_the_future.md) 中探讨其他方法。

#### 处理错误和中止

**[无主复制](./5_replication.md#无主复制)** 的数据存储，主要是在 “尽力而为” 的基础上进行工作：

- **从错误中恢复是应用程序的责任**：数据库将做尽可能多的事，运行遇到错误时，它不会撤消它已经完成的事情；

错误发生不可避免，但许多软件开发人员倾向于只考虑乐观情况，而不是错误处理的复杂性：

- 像 Rails 的 ActiveRecord 和 Django 这样的 **对象关系映射（ORM, object-relation Mapping）** 框架不会重试中断的事务；
  - 从堆栈向上传播的异常，用户拿到一个错误信息
- 一般系统都是中止事务，并返回异常，而不是进行重试，是否能够重试需要在应用层做判断（见下面的理由）；

尽管**重试一个中止的事务是一个简单而有效的错误处理机制**，但它**并不完美**：

- **网络故障导致执行多次**：事务执行成功但服务器跟客户端确认时网络故障，客户端认为提交失败，进行重试，导致执行两次；
- **负载过高**：如果错误是由于负载过大造成的，则重试事务将使问题变得更糟，需要限制重试次数（指数退避算法）；
- **仅临时性错误需要重试**：死锁，异常情况，临时性网络中断和故障切换等错误才值得重试，永久性错误（例如，违反约束）无意义；
- **外部系统的副作用**：事务在数据库之外有副作用，如果确保多个不同的系统一起提交或放弃，使用[两阶段提交](./9_consistency_consensus.md#原子提交与两阶段提交)；

## 弱隔离级别

当一个事务读取由另一个事务同时修改的数据时，或者当两个事务试图同时修改相同的数据时，并发问题（竞争条件）才会出现。

- 并发 BUG 在特殊时序下才会触发，通常很难重现[^4]。

数据库一直试图通过提供 **事务隔离（transaction isolation）** 来隐藏应用程序开发者的并发问题。

- **可串行的（serializable）** 隔离：保证事务的效果如同串行运行一样；
- **可串行的隔离** 会有性能损失，许多数据库不愿意支付这个代价$^{[15]}$；

系统通常使用较弱的隔离级别来防止一部分，而不是全部的并发问题。

- 这些隔离级别难以理解，并且会导致微妙的错误，但是它们仍然在实践中被使用$^{[16]}$
- 

[^1]: 乔・海勒斯坦（Joe Hellerstein）指出，在 Härder 与 Reuter 的论文中，“ACID 中的 C” 是被 “扔进去凑缩写单词的”$^{[5]}$，而且那时候大家都不怎么在乎一致性。
[^2]:这并不完美。如果 TCP 连接中断，则事务必须中止。如果中断发生在客户端请求提交之后，但在服务器确认提交发生之前，客户端并不知道事务是否已提交。为了解决这个问题，事务管理器可以通过一个唯一事务标识符来对操作进行分组，这个标识符并未绑定到特定 TCP 连接。后续再 “[数据库的端到端原则](./12_the_future.md#数据库的端到端原则)” 一节将回到这个主题。

[^3]: 严格地说，**原子自增（atomic increment）** 这个术语在多线程编程的意义上使用了原子这个词。在 ACID 的情况下，它实际上应该被称为 **隔离的（isolated）** 的或 **可串行的（serializable）** 的增量。但这就太吹毛求疵了。
[^4]:  轶事：偶然出现的瞬时错误有时称为 ***Heisenbug***，而确定性的问题对应地称为 ***Bohrbugs***



## 参考文献

1. John D. Cook: “[ACID Versus BASE for Database Transactions](http://www.johndcook.com/blog/2009/07/06/brewer-cap-theorem-base/),” *johndcook.com*, July 6, 2009.
2. Gavin Clarke: “[NoSQL's CAP Theorem Busters: We Don't Drop ACID](http://www.theregister.co.uk/2012/11/22/foundationdb_fear_of_cap_theorem/),” *theregister.co.uk*, November 22, 2012.
3. Peter Bailis, Alan Fekete, Ali Ghodsi, et al.: “[HAT, not CAP: Towards Highly Available Transactions](http://www.bailis.org/papers/hat-hotos2013.pdf),” at *14th USENIX Workshop on Hot Topics in Operating Systems* (HotOS), May 2013.
4. Armando Fox, Steven D. Gribble, Yatin Chawathe, et al.: “[Cluster-Based Scalable Network Services](http://www.cs.berkeley.edu/~brewer/cs262b/TACC.pdf),” at *16th ACM Symposium on Operating Systems Principles* (SOSP), October 1997.
5. Theo Härder and Andreas Reuter: “[Principles of Transaction-Oriented Database Recovery](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.2812&rep=rep1&type=pdf),” *ACM Computing Surveys*, volume 15, number 4, pages 287–317, December 1983. [doi:10.1145/289.291](http://dx.doi.org/10.1145/289.291)
6. Philip A. Bernstein, Vassos Hadzilacos, and Nathan Goodman: [*Concurrency Control and Recovery in Database Systems*](http://research.microsoft.com/en-us/people/philbe/ccontrol.aspx). Addison-Wesley, 1987. ISBN: 978-0-201-10715-9, available online at *research.microsoft.com*.
7. Mai Zheng, Joseph Tucek, Feng Qin, and Mark Lillibridge: “[Understanding the Robustness of SSDs Under Power Fault](https://www.usenix.org/system/files/conference/fast13/fast13-final80.pdf),” at *11th USENIX Conference on File and Storage Technologies* (FAST), February 2013.
8. Laurie Denness: “[SSDs: A Gift and a Curse](https://laur.ie/blog/2015/06/ssds-a-gift-and-a-curse/),” *laur.ie*, June 2, 2015.
9. Adam Surak: “[When Solid State Drives Are Not That Solid](https://blog.algolia.com/when-solid-state-drives-are-not-that-solid/),” *blog.algolia.com*, June 15, 2015.
10. Thanumalayan Sankaranarayana Pillai, Vijay Chidambaram, Ramnatthan Alagappan, et al.: “[All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications](http://research.cs.wisc.edu/wind/Publications/alice-osdi14.pdf),” at *11th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), October 2014.
11. Chris Siebenmann: “[Unix's File Durability Problem](https://utcc.utoronto.ca/~cks/space/blog/unix/FileSyncProblem),” *utcc.utoronto.ca*, April 14, 2016.
12. Lakshmi N. Bairavasundaram, Garth R. Goodson, Bianca Schroeder, et al.: “[An Analysis of Data Corruption in the Storage Stack](http://research.cs.wisc.edu/adsl/Publications/corruption-fast08.pdf),” at *6th USENIX Conference on File and Storage Technologies* (FAST), February 2008.
13. Bianca Schroeder, Raghav Lagisetty, and Arif Merchant: “[Flash Reliability in Production: The Expected and the Unexpected](https://www.usenix.org/conference/fast16/technical-sessions/presentation/schroeder),” at *14th USENIX Conference on File and Storage Technologies* (FAST), February 2016.
14. Don Allison: “[SSD Storage – Ignorance of Technology Is No Excuse](https://blog.korelogic.com/blog/2015/03/24),” *blog.korelogic.com*, March 24, 2015.
15. Peter Bailis, Alan Fekete, Ali Ghodsi, et al.: “[HAT, not CAP: Towards Highly Available Transactions](http://www.bailis.org/papers/hat-hotos2013.pdf),” at *14th USENIX Workshop on Hot Topics in Operating Systems* (HotOS), May 2013.
16. Martin Kleppmann: “[Hermitage: Testing the 'I' in ACID](http://martin.kleppmann.com/2014/11/25/hermitage-testing-the-i-in-acid.html),” *martin.kleppmann.com*, November 25, 2014.