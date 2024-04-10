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

Mini Lab

- [M1：打印进程树](M1_print_the_process_tree.md)
- [M2：最长公共子序列的并行化](./M2_plcs.md)
- [M3：系统调用 Profiler](M3_sperf.md)

- [M4：C语言的REPL](./M4_crepl.md)
- [M5：文件修复](./M5_filerecovery.md)

OS Lab

- [L0]()
- 



### [AbstractMachine: 抽象计算机](https://jyywiki.cn/OS/AbstractMachine/)

> https://git.nju.edu.cn/jyy/os-workbench

提供**运行 C 程序的框架代码和库**。裸机上的 C 语言运行环境，提供 5 组 (15 个) 主要 API，可以实现各类系统软件 (如操作系统)：

- (TRM) `putch`/`halt` - 最基础的计算、显示和停机
- (IOE) `ioe_read/ioe_write` - I/O 设备管理
- (CTE) `ienabled`/`iset`/`yield`/`kcontext` - 中断和异常
- (VME) `protect`/`unprotect`/`map`/`ucontext` - 虚存管理
- (MPE) `cpu_count`/`cpu_current`/`atomic_xchg` - 多处理器



现代线程库：

- Pthread 线程库；
- thread C++ 2011 标准；



- [多处理器间即时可见性的丧失](./2_multi_thread/mem_ordering_test.c)：CPU的指令重排序优化，内存同步障解决问题

  - 单个处理器把汇编代码“编译”成更小的$\mu ops$，每个$\mu ops$ 都有 Fetch, Issue, Execute, Commit 四个阶段
    - “多发射”：每一周期向池子补充尽可能多的 $\mu op$
    - “乱序执行”、“按序提交”：每一周期 (在不违反编译正确性的前提下) 执行尽可能多的 $\mu op$ 

  

[Helgrind: a thread error detector](https://valgrind.org/docs/manual/hg-manual.html)

- 检测C、C ++和Fortran程序中使用符合POSIX标准的线程函数造成的同步错误



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



[A futex overview and update](https://lwn.net/Articles/360699/) (LWN)

[Futexes are tricky](https://jyywiki.cn/pages/OS/manuals/futexes-are-tricky.pdf) (论 model checker 的重要性)



## 生产者/消费者(Leader/follower)

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
