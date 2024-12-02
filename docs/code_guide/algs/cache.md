# 缓存算法

## Window-TinyLFU

LFU（Least Frequecy Use）的缺点：

- 为每一个缓存项维护相应的频率统计信息，一定的存储和性能开销；
- 无法处理稀疏流量（sparse burst）的场景，稀疏流量缓存项频繁被淘汰无法命中；

### `Count-Min Sketch`频率统计算法

**bloom的变形，对元素进行计数**。

- 四个hash函数，其hash值对应计数器+1，且计数器只有4bit（最大15）
- 因存在positive false问题，缓存项的频率值取四个计数器的**最小值**
- 当所有计数器值的和超过设定的阈值（默认是缓存项最大数量的10倍）时，所有的计数器值都减到原来的一半。

位置计算逻辑：`long[] table`数组，大小为缓存空间容量（缓存项最大数量）向上取整为2的n次方

- 一个 **long 分为 4 个 group**；
- hash 值计算数组下标，根据值低 2 bit 确定 group，**第 X 个计数器位置为对应 group 的第X个位置**；

<img src=".pics/cache/cmsketch.webp" alt="Count-Min Sketch" style="zoom: 67%;" />

### W-TinyLFU 算法

将缓存存储空间分为两个大的区域：

- **Window Cache**：标准的 LRU cache；
- **Main Cache**：SLRU（Segmemted LRU） cache；
  - 进一步划分为 Protected Cache（保护区）和 Probation Cache（观察区）两个区域
  - Protected：缓存项不会被淘汰
  - Probation：根据LRU进行淘汰

<img src=".pics/cache/tinylfu.webp" alt="img" style="zoom:67%;" />

流程：

- 有新的缓存项写入缓存时，会先写入Window Cache区域
- 当Window Cache空间满时，最旧的缓存项会被移出Window Cache
- 如果Probation Cache未满，从Window Cache移出的缓存项会直接写入Probation Cache
- 如果Probation Cache已满，则会根据TinyLFU算法确定从Window Cache移出的缓存项是丢弃（淘汰）还是写入Probation Cache
- Probation Cache中的缓存项如果访问频率达到一定次数，会提升到Protected Cache
- 如果Protected Cache也满了，最旧的缓存项也会移出Protected Cache，然后根据TinyLFU算法确定是丢弃（淘汰）还是写入Probation Cache。



TinyLFU（即4bits计数器的Count-Min Sketch）淘汰机制为：图中的Candidate和Victim

- 如果Candidate缓存项的访问频率大于Victim缓存项的访问频率，则淘汰掉Victim；
- 如果Candidate小于或等于Victim的频率，且如果Candidate的频率小于5，则淘汰掉Candidate；
- 否则，则在Candidate和Victim两者之中随机地淘汰一个；



**特性**

- 最近刚产生的缓存项进入Window区，不会被淘汰；
- 访问频率高的缓存项进入Protected区，也不会淘汰；