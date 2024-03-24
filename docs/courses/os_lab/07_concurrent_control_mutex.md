# 并发控制：互斥

**本讲内容**：现代多处理器系统上的互斥实现：

- 互斥问题的定义和假设
- 自旋锁
- 互斥锁和 Futex

## 互斥问题：定义与假设

### 互斥问题：定义

> Lock 保护临界区，不同的 Lock 是并发的，是如何实现的？
>
> - 临界区的变量是如何区分可以并发操作的？汇编指令是啥？
> - 如果每次都是要将数据写内存/读内存，那么不同的锁之间也会有影响？

互斥 (mutual exclusion)，“互相排斥”

- 实现 `lock_t` 数据结构和 `lock/unlock` API:

```c
typedef struct {
  ...
} lock_t;
void lock(lock_t *lk);
void unlock(lock_t *lk);
```

一把 “排他性” 的锁——对于锁对象 `lk`

- 如果某个线程持有锁，则其他线程的 `lock` 不能返回 (Safety)
- 在多个线程执行 `lock` 时，至少有一个可以返回 (Liveness)
- 能<font color='red'>正确处理处理器乱序、宽松内存模型和编译优化</font>

### 实现互斥的基本假设

允许使用使我们可以不管一切麻烦事的原子指令

```c
void atomic_inc(long *ptr);
int atomic_xchg(int val, int *ptr);
```

看起来是一个普通的函数，但假设：

- **原子性**：包含一个原子指令
  - 指令的执行不能被打断
- **执行顺序**：包含一个 compiler barrier
  - 无论何种优化都不可越过此函数
- **可见性**：包含一个 memory fence
  - 保证处理器在 stop-the-world 前所有对内存的 store 都 “生效”
  - 即对 resume-the-world 之后的 load 可见

### Atomic Exchange 实现

```c
int xchg(int volatile *ptr, int newval) {
  int result;
  asm volatile(
    // 指令自带 memory barrier
    "lock xchgl %0, %1"
    : "+m"(*ptr), "=a"(result)
    : "1"(newval)
    // Compiler barrier
    : "memory"
  );
  return result;
}
```



## 自旋锁（Spin Lock） 

科学家：考虑更多更根本的问题

- 我们可以设计出怎样的原子指令？
  - 它们的表达能力如何？
- 计算机硬件可以提供比 “一次 load/store” 更强的原子性吗？
  - 如果硬件很困难，软件/编译器可以么？

### 自旋锁：用 `xchg` 实现互斥

