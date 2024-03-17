# OS Lab 2023

> 官方网站：https://jyywiki.cn/OS/2023/
>
> 参考书籍：
>
> - [The Hitchhiker’s Guide to Operating Systems](./pdfs/The Hitchhiker’s Guide to Operating Systems.pdf)
> - [Parallel and Distributed Computation: Numerical Methods](https://web.mit.edu/dimitrib/www/pdc.html)
>
> 参考书：
>
> - [OSTEP] Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau. *[Operating Systems: Three Easy Pieces](http://pages.cs.wisc.edu/~remzi/OSTEP/)*. Arpaci-Dusseau Books, 2018.
> - [CSAPP] Randal E. Bryant and David R. O'Hallaron. *Computer Systems: A Programmer's Perspective* (3ed). Pearson, 2017. (作为手册和参考书查阅)
>
> 



## 课程大纲

> 随堂（课件中的）代码：[OS_LAB_CODE](https://gitee.com/oscsc/oslabcode)
>
> MiniLab 和 OSLab 应[要求](https://jyywiki.cn/OS/2023/labs/Labs.html)，不进行开源。

1.  [操作系统概述](./01-os_introduction.md)
2.  [应用视角的操作系统](./02-os_appview.md)
3.  [硬件视角的操作系统](03-os_hardware_view.md)
4.  [操作系统模型](04-os_python_modeling.md)

[M1: 打印进程树 (pstree)](./M1_print_the_process_tree.md)

L0: 为计算机硬件编程 



20. [C标准库和实现](./20-c_standard_lib.md)
21. [可执行文件与加载](./21-22-executable_files_loading.md)

#### 





## 课后实验

> 

M1：打印进程树





### [AbstractMachine: 抽象计算机](https://jyywiki.cn/OS/AbstractMachine/)

> https://git.nju.edu.cn/jyy/os-workbench

提供**运行 C 程序的框架代码和库**。裸机上的 C 语言运行环境，提供 5 组 (15 个) 主要 API，可以实现各类系统软件 (如操作系统)：

- (TRM) `putch`/`halt` - 最基础的计算、显示和停机
- (IOE) `ioe_read/ioe_write` - I/O 设备管理
- (CTE) `ienabled`/`iset`/`yield`/`kcontext` - 中断和异常
- (VME) `protect`/`unprotect`/`map`/`ucontext` - 虚存管理
- (MPE) `cpu_count`/`cpu_current`/`atomic_xchg` - 多处理器



## 2. 多处理器编程

> **“程序 (甚至是一条指令) 独占处理器执行”** 的基本假设在现代多处理器系统上不再成立。
>
> **不原子、能乱序、不立即可见**

(历史) 1960s，大家争先在共享内存上实现原子性 (互斥)

- 但几乎所有的实现都是错的，直到 [Dekker's Algorithm](https://en.wikipedia.org/wiki/Dekker's_algorithm)，还只能保证两个线程的互斥

现代线程库：

- Pthread 线程库；
- thread C++ 2011 标准；



**编译器做的是静态乱序优化，CPU做的是动态乱序优化。**

```c
// volatile 关键字表示每次内存数据读取都走主存，避免静态乱序优化
volatile 
// 函数表示内存屏障/禁止指令重排序，避免动态乱序优化
__sync_synchronize()
```



### 课程代码

- [课程提供的线程库](./2_multi_thread/thread.h)
- [线程共享内存测试](./2_multi_thread/shm_test.c)
- [线程独立堆栈和范围测试](./2_multi_thread/stack_test.c)
- [多线程竞态测试](./2_multi_thread/atomic_test.c)：单个处理器的乱序执行，编译器的重排序优化
  - `-O`优化：**编译器对内存访问 “eventually consistent” 的处理导致**共享内存作为线程同步工具的失效。
- [多处理器间即时可见性的丧失](./2_multi_thread/mem_ordering_test.c)：CPU的指令重排序优化，内存同步障解决问题
  - 单个处理器把汇编代码“编译”成更小的$\mu ops$，每个$\mu ops$ 都有 Fetch, Issue, Execute, Commit 四个阶段
    - “多发射”：每一周期向池子补充尽可能多的 $\mu op$
    - “乱序执行”、“按序提交”：每一周期 (在不违反编译正确性的前提下) 执行尽可能多的 $\mu op$ 
- [Peterson's 协议](./2_multi_thread/peterson_simple.c):  需要避免指令重排序
  - Sequential 内存模型下 Peterson's Protocol 的 Safety

- [顺序内存模型的状态机检查](./2_multi_thread/model_check/model_check.py)：仅适用于顺序内存模型（即顺序执行指令）



[Helgrind: a thread error detector](https://valgrind.org/docs/manual/hg-manual.html)

- 检测C、C ++和Fortran程序中使用符合POSIX标准的线程函数造成的同步错误

### 参考资料

 [ARM/RISC-V 内存模型](https://research.swtch.com/mem-weak@2x.png)

[X86内存模型](https://research.swtch.com/hwmm)

[Ad hoc synchronization considered harmful](https://www.usenix.org/events/osdi10/tech/full_papers/Xiong.pdf) 

[Model checking for programming languages using VeriSoft](https://dl.acm.org/doi/abs/10.1145/263699.263717)：第一个 “software model checker”，不记状态；

[Finding and reproducing Heisenbugs in concurrent programs](https://dl.acm.org/doi/10.5555/1855741.1855760)

[Using model checking to find serious file system errors](https://dl.acm.org/doi/10.1145/1189256.1189259)

[VSync: Push-button verification and optimization for synchronization primitives on weak memory models](https://dl.acm.org/doi/abs/10.1145/3445814.3446748)



TODO: model_check的代码

### M2: 协程库 (libco)

> 

## 3. 并发编程（互斥）

> 解决问题的两种方法：
>
> 1）提出算法，解决问题（Dekker/Peterson/...'s Protocols）；
>
> 2）改变假设（软件不够，硬件来凑（自旋锁）；用户不够，内核来凑 (互斥锁)）；

硬件能为我们提供一条 “瞬间完成” 的读 + 写指令 : [stdatomic.h](https://en.cppreference.com/w/cpp/header/stdatomic.h)

- `xchg`：原子性的交换

### 自旋锁（Spin Lock）

#### x86 原子操作：`LOCK` 指令前缀

```c
int xchg(volatile int *addr, int newval) {
  int result;
  asm volatile ("lock xchg %0, %1" : "+m"(*addr), "=a"(result) : "1"(newval));
  return result;
}
```

**原子指令的模型**（不触发系统调用）

- 保证之前的 store 都写入内存
- 保证 load/store 不与原子指令乱序

#### Lock 指令的现代实现

在 **L1 cache 层保持一致性** (ring/mesh bus)

- 相当于每个 cache line 有分别的锁
- `store(x)` 进入 **L1 缓存即保证对其他处理器可见**
  - 但要小心 **store buffer** 和**乱序执行**

L1 cache line 根据状态进行协调

- **M (Modified)**， 脏值
- **E (Exclusive)**，独占访问
- **S (Shared)**，只读共享
- **I (Invalid)**，不拥有 cache line

**Load-Reserved/Store-Conditional (LR/SC)**：RISC-V， 另一种原子操作的设计

- LR: 在内存上标记 reserved (盯上你了)，中断、其他处理器写入都会导致标记消除；
- SC: 如果 “盯上” 未被解除，则写入。
- 硬件实现：BOOM (Berkeley Out-of-Order Processor) [`lsu/dcache.scala`](https://github.com/riscv-boom/riscv-boom/blob/master/src/main/scala/lsu/dcache.scala#L655)
  - 留意 s2_sc_fail 的条件s2 是流水线 Stage 2

性能问题：

- 自旋 (共享变量) 会触发处理器间的缓存同步，延迟增加
- 除了进入临界区的线程，其他处理器上的线程都在空转，争抢锁的处理器越多，利用率越低
- 获得自旋锁的线程**可能被操作系统切换出去**

#### 自旋锁的使用场景

- **临界区几乎不 “拥堵”**

- 持有自旋锁时**禁止执行流切换**


使用场景：**操作系统内核的并发数据结构 (短临界区)**

- 操作系统可以**关闭中断和抢占**
  - 保证锁的持有者在很短的时间内可以释放锁
- (如果是虚拟机呢...😂)
  - PAUSE 指令会触发 VM Exit
- 但依旧很难做好
  - [An analysis of Linux scalability to many cores](https://www.usenix.org/conference/osdi10/analysis-linux-scalability-many-cores) (OSDI'10)

### 互斥锁 （Mutex Lock）

#### 实现线程 + 长临界区的互斥

“让” 不是 C 语言代码可以做到的 (C 代码只能计算)，**把锁的实现放到操作系统里**就好啦！

- `syscall(SYSCALL_lock, &lk);`
  - 试图获得 `lk`，但如果失败，就切换到其他线程
- `syscall(SYSCALL_unlock, &lk);`
  - 释放 `lk`，如果有等待锁的线程就唤醒



### Futex: Fast Userspace muTexes

#### 自旋锁/睡眠锁对比分析

自旋锁 (线程直接共享 locked)

- 更快的 fast path
  - xchg 成功 → 立即进入临界区，开销很小
- 更慢的 slow path
  - xchg 失败 → 浪费 CPU 自旋等待

睡眠锁 (通过系统调用访问 locked)

- 更快的 slow path
  - 上锁失败线程不再占用 CPU
- 更慢的 fast path
  - 即便上锁成功也需要进出内核 (syscall)

#### Futex = Spin + Mutex

> 性能优化的最常见技巧：看 average (frequent) case 而不是 worst case

POSIX 线程库中的互斥锁 (pthread_mutex)

- Fast path: 一条原子指令，上锁成功立即返回

- Slow path: 上锁失败，执行系统调用睡眠



[A futex overview and update](https://lwn.net/Articles/360699/) (LWN)

[Futexes are tricky](https://jyywiki.cn/pages/OS/manuals/futexes-are-tricky.pdf) (论 model checker 的重要性)



## 3. 并发编程（同步）

### 生产消费

> 生产-消费模型，能够解决99.9%的并发问题。

示例要求：

```c
void Tproduce() { while (1) printf("("); }
void Tconsume() { while (1) printf(")"); }
```

在 `printf` 前后增加代码，使得打印的括号序列满足

- 一定是某个合法括号序列的前缀
- 括号嵌套的深度不超过 *n*

#### 互斥锁

```c
void Tconsume() {
  while (1) {
retry:
    mutex_lock(&lk);
    if (count == 0) {
      mutex_unlock(&lk);
      // spin
      goto retry;
    }
    count--;
    printf(")");
    mutex_unlock(&lk);
  }
}
```

#### 条件变量

自旋 => 睡眠

```c
void Tconsume() {
  while (1) {
    // 条件变量的万能公式
    mutex_lock(&lk);
    while (count == 0) {
      cond_wait(&cv, &lk);
    }
    printf(")"); count--;
    cond_broadcast(&cv);
    mutex_unlock(&lk);
  }
}
```

#### 信号量

```c
void consumer() {
  while (1) {
    P(&fill);
    printf(")");
    V(&empty);
  }
}
```



### 哲学家问题

哲学家 (线程) 有时思考，有时吃饭

- 吃饭需要同时得到左手和右手的叉子
- 当叉子被其他人占有时，必须等待，如何完成同步？

#### 如何用互斥锁/信号量实现？

成功的尝试 (万能的方法)

```c
mutex_lock(&mutex);
while (!(avail[lhs] && avail[rhs])) {
  wait(&cv, &mutex);
}
avail[lhs] = avail[rhs] = false;
mutex_unlock(&mutex);

mutex_lock(&mutex);
avail[lhs] = avail[rhs] = true;
broadcast(&cv);
mutex_unlock(&mutex);
```

### 生产者/消费者(Leader/follower)

- 分布式系统中非常常见的解决思路 (HDFS, ...)

```c
void Tphilosopher(int id) {
  send_request(id, EAT);
  P(allowed[id]); // waiter 会把叉子递给哲学家
  philosopher_eat();
  send_request(id, DONE);
}

void Twaiter() {
  while (1) {
    (id, status) = receive_request();
    if (status == EAT) { ... }
    if (status == DONE) { ... }
  }
}
```

抛开 workload 谈优化就是耍流氓

- 吃饭的时间通常远远大于请求服务员的时间
- 如果一个 manager 搞不定，可以分多个 (fast/slow path)
  - 把系统设计好，使集中管理不成为瓶颈
    - [Millions of tiny databases](https://www.usenix.org/conference/nsdi20/presentation/brooker) (NSDI'20)
## 3. 并发编程
《真实世界的并发编程》
1. 大规模并行（HPC，大数据）、高性能并发（数据中心）
2. 01:00:00 go & go routine
3. 01:34 单线程+事件模型（异步回调，流程模型）

《并发BUG与应对》
并发bug: 防御式编程，使用`assert`

死锁：AA-Deadlock，ABBA-Deadlock
1. 死锁的必要条件；
2. 避免死锁；
   - AA-Deadlock：容易检测，及早报告；`spinlock-xv6.c`, `if (holding(lk) panic();)
   - ABBA-Deadlock：按照固定的顺序去获得锁，相反顺序释放锁；


数据竞争：
- Atomic Violation / Order Violation

运行时的死锁检查（Lockdep) 
- 为每个锁确定唯一的“allocation site"，同一个"allocation site"的存在唯一的分配顺序；
- 通过打印上锁顺序，判断是否存在环（ x->y ^ y->x ）

运行时的数据竞争（Thread Sanitizer）
- 为所有事件建立 happens-before 关系；
- Program-order + release-accquire
- 对于发生在不同线程且至少有一个是写的x,y检查： x ~ y V y ~ x
  - Times, clocks ......


4. 操作系统状态机
