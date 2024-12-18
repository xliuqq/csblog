# List

## 链表

单链表，双链表

- 判断单向链表中是否有环：快慢指针算法；

## 跳表(SkipList)

> 跳表（Skiplist）是一个特殊的链表，相比一般的链表，有更高的查找效率，可比拟二叉查找树，平均期望的查找、插入、删除时间复杂度都是O(logn)。

- 建立层级索引，减少比较的次数，增加搜索效率；
- 空间增加一倍；

### 示意图

![skip_list](.pics/list/skip_list.jpg)

### 算法流程

1. 新节点和各层索引节点逐一比较，确定原链表的插入位置。O（logN）
2. 把索引插入到原链表。O（1）
3. 利用抛硬币的随机方式，决定新节点是否提升为上一级索引。结果为“正”则提升并继续抛硬币，结果为“负”则停止。O（logN）



## RCU无锁链表

> **在更新的过程中，运行多个reader进行读操作，写少读多的场景**。
>
> **一个写多个读**的同步机制，而不是多个写多个读的同步机制（多写时需要加锁）。

- RCU是由`RCU-sync`和`COW(Copy-On-Write)`两部分组成的，Java中有GC因此只有COW；
- **`RCU-sync`**：解决拷贝出来的对象什么时候回收的问题（核心部分）；

典型的RCU**更新时序**如下:

- **复制**：将需要更新的数据复制到新内存地址；

- **更新**：更新复制数据，这时候操作的新的内存地址；

- **替换**：使用新内存地址指针替换旧数据内存地址指针

此后旧数据将无法被后续读者访问；

- **等待**，所有访问旧数据的读者进入静默期，即访问旧数据完成；

- **回收**：当没有任何持有旧数据结构引用的读者后，安全地回收旧数据内存。



### 读操作

- 直接对共享资源进行访问，现代`CPU`支持访存操作的原子化；
- **读操作上下文是不可抢占的**，读访问共享资源时可以采用`read_rcu_lock()`，该函数的工作是停止抢占；



### 写操作

- 复制 -> 更新 -> 替换：替换旧数据的指针时，**需要内存屏障**，避免指令重排序；



### 垃圾回收

`daemon`，当共享资源被`update`之后，可以采用该`daemon`实现老数据资源的回收：

- 在`update`之前的所有的读者全部退出；
- **写者在update之后是需要睡眠等待的**，需要等待读者完成操作，如果在这个时刻**读者被抢占或者睡眠，那么很可能会导致系统死锁**；

**宽限期**（Grace period：在读取过程中，另外一个线程删除一个节点。删除线程可以把这个节点从链表中移除，但它不能直接销毁这个节点，必须等到所有的读取线程读取完成以后，才进行销毁操作。

- 宽限期的意义是，在一个删除动作发生后，它必须等待所有在宽限期开始前已经开始的读线程结束，才可以进行销毁操作



### API

- `rcu_read_lock`和`rcu_read_unlock`必须是成对出现的，并且`synchronize_rcu`不能出现在`rcu_read_lock`和`rcu_read_unlock`之间；

```c++
// lock, unlock 帮助检测宽限期是否结束
#define rcu_read_lock() __rcu_read_lock()
#define rcu_read_unlock() __rcu_read_unlock()
#define __rcu_read_lock() 
	do { 
		preempt_disable(); 
		__acquire(RCU); 
		rcu_read_acquire(); 
	} while (0)

#define __rcu_read_unlock() 
	do { 
		rcu_read_release(); 
  	  	__release(RCU); 
   	 	preempt_enable(); 
    } while (0)
```

- RCU**同时只允许一个`synchronize_rcu`操作**，所以需要我们自己来实现synchronize_rcu的排它锁操作；

```c
// 调用该函数意味着一个宽限期的开始，而直到宽限期结束，该函数才会返回
// 通过阻塞来做到这一点，直到所有cpu上所有预先存在的RCU读端临界区都完成
synchronize_rcu()
```

- `rcu_assign_pointer`修改旧数据的指针；

```c
// 新数据
#define rcu_assign_pointer(p, v) __rcu_assign_pointer((p), (v), __rcu)

#define __rcu_assign_pointer(p, v, space) 
    do { 
        smp_wmb();  // 内存屏蔽障

        (p) = (typeof(*v) __force space *)(v); 
    } while (0)
```

- 

```c
// 用于读临界区
rcu_dereference
```



示例：

```c
rcu_read_lock();
list_for_each_entry_rcu(pos, head, member) {
    // do something with `pos`
}
rcu_read_unlock();

/* p 指向一块受 RCU 保护的共享数据 */
/* reader */
rcu_read_lock();
p1 = rcu_dereference(p);
if (p1 != NULL) {
    printk("%d\n", p1->field);
}
rcu_read_unlock();

/* free the memory */
p2 = p;
if (p2 != NULL) {
    p = NULL;
    synchronize_rcu();
    kfree(p2);
}
```



