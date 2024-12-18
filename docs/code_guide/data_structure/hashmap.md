# Hash 表

哈希表是根据设定的哈希函数H（key）和处理冲突方法将**一组关键字**映射到一个**有限的地址区间**上，并以关键字在地址区间中的象作为记录在表中的存储位置，这种表称为哈希表或散列，所得存储位置称为哈希地址或散列地址。



## 热点Hash

> [1] [Chen J ,  Chen L ,  Wang S , et al. HotRing: a hotspot-aware in-memory key-value store[C]// File and Storage Technologies. 2020.](./pdf/a hotspot-aware in-memory key-value store.pdf)

内存KVS引擎通常采用**链式哈希**作为索引，访问位于冲突链尾部的数据，需要经过更多的索引跳数：

- **将热点数据放置在冲突链头部**，那么系统对于热点数据的访问将会有更快的响应速度。

### 有序环式哈希索引

- 将冲突链首尾连接形式冲突环，保证头指针指向任何一个item都可以遍历环上所有数据
- 通过**lock-free移动头指针**，动态指向热度较高的item（或根据算法计算出的最优item位置）

<img src=".pics/hashmap/hotring.png" alt="图片" style="zoom: 50%;" />

#### 有序环

**在环上，所有数据都会动态变化(更新或删除)，头指针同样也会动态移动，没有标志可以作为遍历的终止判断。**

- key排序：若目标key介于连续两个item的key之间，说明为read miss操作，即可终止返回；
- key简化：利用tag来减少key的比较开销，字典序：order = (tag, key)
  - tag是哈希值的一部分，每个key计算的哈希值，前k位用来哈希表的定位，后n-k位作为tag；

<img src=".pics/hashmap/order_hash_compare.jpeg" alt="图片" style="zoom: 50%;" />

以 itemB 举例：

- 链式哈希需要遍历所有数据才能返回read miss。而HotRing在访问itemA与C后，即可确认B read miss。

#### 动态识别与调整

每R次访问为一个周期(R通常设置为5)，第R次访问的线程将进行头指针的调整：

- **随机移动策略**：每R次访问，移动头指针指向第R次访问的item。若已经指向该item，则头指针不移动。该策略的优势是， 不需要额外的元数据开销，且不需要采样过程，响应速度极快。

- **采样分析策略**：每R次访问，尝试启动对应冲突环的采样，统计item的访问频率。若第R次访问的item已经是头指针指向的item，则不启动采样。

采样分析策略：利用了head pointer / next pointer的剩余空间，因为指示地址用48位就够了，这样还有16位：

- head pointer：16位分成1 bit的active位以及15位的total counter位（记录当前这个bucket的访问次数）；
- next pointer：1 bit Rehash位，1bit Occupied位和14 bit的counter位，Rehash和Counter分别用于并发的rehash和update操作。

<img src=".pics/hashmap/hotring_index.png" alt="在这里插入图片描述" style="zoom:50%;" />


TODO：[性能提升2.58倍！阿里最快KV存储引擎揭秘 (qq.com)](https://mp.weixin.qq.com/s/0IKxbt8MDH6Yqu1f00cwSA)



#### 无锁并发访问

修改和更新并发正确性问题：

- 在链A->B->D上，线程1进行插入C的操作，同时线程2进行RCU更新B的操作，尝试更新为B'。线程1修改B的指针指向C，完成插入。而线程2修改A的指针指向B'完成更新。**两个线程并发修改不同的内存，均可成功返回。**但是这时遍历整条链(A->B'->D)，将发现C无法被遍历到，导致正确性问题。

解决措施

- 利用Item的**Occupied**标志位。当线程2更新B时，首先需要**将B的Occupied标志位置位**。线程1插入C需要修改B的指针(Next Item Address)，若发现Occupied标志位已置位，**则需要重新遍历链表，尝试插入**。通过使并发操作竞争修改同一内存地址，保证并发操作的正确性。

<img src=".pics/hashmap/lock-free_list_insert.png" alt="图片" style="zoom:67%;" />



#### 适应热点数据量的无锁rehash

引擎性能的降低，必然是因为冲突环上存在多个热点。

- 利用访问所需**平均内存访问次数**(access overhead)来替代传统rehash策略的负载因子(load factor)。

在幂率分布场景，若每个冲突环只有一个热点，HotRing可以保证access overhead < 2，即平均每次访问所需内存访问次数小于2。

- 设定access overhead阈值为2，当大于2时，触发rehash。



rehash过程分为3步进行：图一为哈希表，哈希值在rehash前后的变化。剩余三图为rehash三个过程。

<img src=".pics/hashmap/hotring_rehash-1.png" alt="图片" style="zoom:67%;" />

<img src=".pics/hashmap/hotring_rehash.jpeg" alt="图片" style="zoom:67%;" />

- 初始化：创建后台rehash线程，创建2倍空间的新哈希表，通过复用tag的最高一位来进行索引；
  - 新表中将会有两个头指针与旧表中的一个头指针对应；
  - 假设tag最大值为T，tag范围为[0,T)，则两个新的头指针对应tag范围为[0,T/2)和[T/2,T)；
  - rehash线程创建一个rehash节点(包含两个空数据的子item节点)，子item节点分别对应两个新头指针，利用item中的Rehash标志位识别rehash节点的子item节点。
- 分裂：rehash线程通过将rehash节点的两个子item节点插入环中完成环的分裂。
  - 如图(Split)所示，因为itemB和E是tag的范围边界，所以子item节点分别插入到itemB和E之前。
  - 完成两个插入操作后，新哈希表将激活，所有的访问都将通过新哈希表进行访问；
  - 逻辑上将冲突环一分为二。当我们查找数据时，最多只需要扫描一半的item。
- 删除：收尾工作，包括旧哈希表的回收和rehash节点的删除回收。

分裂阶段和删除阶段间，必须有一个**RCU静默期(transition period)**。

- 静默期保证所有**从旧哈希表进入的访问均已经返回**。否则，直接回收旧哈希表可能导致并发错误。

