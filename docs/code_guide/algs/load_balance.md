# 负载均衡算法

处理负载均衡的两种方式：

**静态处理**：Consistent hash，引入virtual nodes实现环状 hash 空间的均匀散列；

**动态处理**：Random、Round-robin、Least connection、**Two random choices**



## The Power of Two Random Choices

> Mitzenmacher M. The power of two choices in randomized load balancing[J]. IEEE Transactions on Parallel and Distributed Systems, 2001, 12(10): 1094-1104.

算法流程是：

1. 从可用节点列表中做两次随机选择操作，得到节点 A、B
2. **比较 A、B 两个节点，选出负载最低（一般是正在处理的连接数/请求数最少）的节点作为被选中的节点**

**Balls into Bins问题**：N个球分配到N个桶

问题1： 每次随机独立选择一个桶，最后，桶里的球的最大个数？

答案：**`1-ｏ(1)`** 的概率，最大个数是**`(1+ｏ(1))log(n)/log(log(n))`**，`o(1)`指的是高阶无穷小。

问题2： 每次随机D次挑出D个桶，并从中选择球个数少的桶呢？

答案：**`1-ｏ(1)`** 的概率，最大个数是**`(1+ｏ(1))log(log(n))/log(d)+Ｏ(1)`**，`o(1)`指的是高阶无穷小。



问题3：m个球，n个桶，每次随机独立选择d个桶，选择最少的桶放入，其中 `m>=n,d>=2`

- n 个桶中最大负载为$\frac{log(log(n))}{log(d)}+\frac{m}{n}+\theta(1)$

### Small Cache, Big Effect

> [Fan B, Lim H, Andersen D G, et al. Small cache, big effect: Provable load balancing for randomly partitioned cluster services[C]//Proceedings of the 2nd ACM Symposium on Cloud Computing. ACM, 2011: 23.](https://www.alipan.com/s/Aatn7Y2a9d8)

在 load balancer 中集成一层空间复杂度下界为 O(nlogn) 的cache*（n为后端节点数）*来缓存**热点数据**，即可在显著提升load balancer吞吐量的同时，保证到达后端节点的请求负载是均匀分布的。

- 需要缓存的热点数据量只与存储节点数目n有关，与存储的数据量无关



### Distcache: Provable load balancing for large-scale storage systems with distributed caching

一个集群一个Cache节点只能保证集群内的负载均衡，而集群间的负载均衡无法保证。

集群间需要一层分布式的Cache。

- 通过两层hash来partition Cache中的数据，保证其一层中有热点的节点会被散列到另外一层。然后通过power-of-two-choices来选择负载较低的节点访问。
- 保证要维护的一致性开销最小化，同时论文理论证明了这个做法是不会导致任何一个Cache引热点数据过载。