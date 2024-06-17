# 状态机模型的应用

**本讲内容**：回顾操作系统对象、API 上是如何构建应用程序世界的：

- 操作系统和应用程序的状态机模型
- 状态机模型的应用

## Everything is a State Machine

### 一切皆为状态机

在课程中状态机模型的变化

- minimal.S
  - 指令和系统调用两种状态迁移
- hanoi.c
  - C 程序也是状态机
- stack-prob.c
  - 共享内存、独立堆栈的状态机
- loader-static.c
  - 通过 mmap 实现地址空间的管理

### 操作系统对象和 API

访问操作系统对象

- 文件描述符：指向操作系统对象的 “指针” (handle)
- fork() 时会继承

状态机管理 API

- 状态机创建与更新：fork, execve, exit
- 地址空间管理：mmap, sbrk

操作系统对象管理 API

- open, read, write, lseek, pipe, ...

## 状态机：建模理解我们的世界

### Conway's Game of Life

[Game of life](https://playgameoflife.com/) (Cellular automata)

- 把世界想象成是一个状态机
- [一个程序如何感知自己是在真机还是虚拟机执行](https://oimo.io/works/life/)？
- 如何理解四维空间 (𝑥,𝑦,𝑧,𝑢)

### 更严肃（理论）的探讨

在状态机模型上严格定义很多概念：预测未来、时间旅行……

- 成为你理解物理 (和计算机) 世界的参考

Cellular automata 不支持 “时间旅行”

- 怎么添加一个公理使它可以支持？
  - 平行宇宙
  - 如果世界线需要合并？可以[收敛于某个分布](https://www.scientificamerican.com/article/time-travel-simulation-resolves-grandfather-paradox/)

Cellular automata 不支持 “预测外来”

- 能否添加一个 syscall 使它支持？
  - [Why philosophers should care about computational complexity, Ch. 10](https://www.scottaaronson.com/papers/philos.pdf)

## ️🌶️ 状态机：建模理解程序的世界

### Trace 和调试器

程序执行 = 状态机执行：“hack” 进这个状态机

- 观察状态机的执行：strace/gdb
- 甚至记录和改变状态机的执行

<font color='red'>**Time-Travel Debugging & Record-replay**</font>

- 如何退回到 $s_0 \rightarrow s_1 \rightarrow s_2 \rightarrow ...$中的任何一个状态？
  - AskGPT: How is time-travel debugging implemented in gdb?
- 我们甚至可以完整记录程序的执行
  - [record_and_replay](https://dl.acm.org/doi/pdf/10.1145/3386277?download=true), QEMU, ...

### 性能优化和 Profiler

> <font color='red'>**Premature optimization is the root of all evil. (D. E. Knuth)**</font>

那到底怎么样才算 mature 呢？

- 状态机的执行需要时间；对象需要占用空间
- 需要理解好 “时间花在哪里”、“什么对象占用了空间”
- 本质的回答：“为了做某件事到底花去了多少资源”
- 简化的回答：“一段时间内资源的消耗情况”

------

![img](pics/safari-profiling.png)

<font color='red'>**真实执行的性能摘要**</font>！

- 思路：隔一段时间 “暂停” 程序、观察状态机的执行：**中断就可以做到**

  - 将状态 *s*→*s*′ “记账”
    - 执行的语句，函数调用栈，服务的请求

  - 得到统计意义的性能摘要

例子：<font color='red'>**Linux Kernel perf **</font>(支持硬件 PMU)

- `perf list, perf stat (-e), perf record, perf report`

<font color='red'>**二八定律：80% 的时间消耗在非常集中的几处代码**</font>

- L1 (pmm): 小内存分配时的 lock contention
  - profiler 直接帮你解决问题

工业界遇到的大部分情况

- 木桶效应：每个部分都已经 tune 到局部最优了
  - 剩下的部分要么 profiler 信息不完整，要么就不好解决
  - [火焰图：The flame graph](https://cacm.acm.org/magazines/2016/6/202665-the-flame-graph/fulltext) (CACM'16)

### Model Checker 和 Verifier

一些真正的 model checkers

- [TLA+](https://lamport.azurewebsites.net/tla/tla.html) by Leslie Lamport;
- [Java PathFinder (JFP)](https://ti.arc.nasa.gov/tech/rse/vandv/jpf/) 和 [SPIN](http://spinroot.com/)
- 如果你希望直接验证<font color='red'>**程序**</font>？

任何 “non-deterministic” 的状态机都可以检查

```c
u32 x = rdrand(), y = rdrand();
if (x > y)
  if (x * x + y * y == 65)
    bug();  // 按值分状态图则分支爆炸，如何有效地检验？按条件
```

- [KLEE: Unassisted and automatic generation of high-coverage tests for complex systems programs](https://dl.acm.org/doi/10.5555/1855741.1855756) (OSDI'08, Best Paper 🏅)



## 课后习题/编程作业

### 1. 阅读材料

教科书 Operating Systems: Three Easy Pieces

- 第 6 章 - [Direct Execution](book_os_three_pieces/06-cpu-mechanisms.pdf)
- 第 7 章 - [CPU Scheduling](book_os_three_pieces/07-cpu-sched.pdf)
- 第 8 章 - [Multi-level Feedback](book_os_three_pieces/08-cpu-sched-mlfq.pdf)
- 第 9 章 - [Lottery Scheduling](book_os_three_pieces/09-cpu-sched-lottery.pdf)
- 第 10 章 - [Multi-CPU Scheduling](book_os_three_pieces/10-cpu-sched-multi.pdf)