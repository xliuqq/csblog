---
date: 2023-10-23
readtime: 10
categories:
  - 并发
tags:
  - 论文
---



# 了解真实世界的GO并发Bugs

> 论文：2019《Understanding Real-World Concurrency Bugs in Go》

Go 推荐使用消息传递（message passing）作为线程（goroutine）间通信的手段，但是**线程间通信采用消息传递还是共享内存同步机制**，并没有真实的对比显式优劣性。

论文做了以下贡献：

- 第一个系统研究真实GO程序（如 docker/k8s/etcd等）的并发bugs：

- 





消息传都跟共享内存一样，很容易出现并发bugs。



<!-- more -->

## Go 并发使用模式分析

### Goroutine 使用

> 观察结论1：**Goruntines 的生命周期更短，但是比 C（线程）创建更频繁**（无论是静态还是运行时创建）

**问题1：Do real Go programmers tend to write their code with many goroutines (static) ?**

**表2：每千行代码 goroutine 的创建数**（gRPC-C指的是线程的创建数）

- 相较于C项目，Go中的各项目中都大量的使用了协程

![goroutine_usage_per_kloc](pics/goroutine_usage_1.png)

**问题2：Do real Go applications create a lot of goroutines during runtime (dynamic) ?**

**表3**：Google RPC Benckmarking 中的**Go和C应用的协程/线程**的值，以及相对于应用程序的协程/线程的占总应用时间

- 动态创建的 Go routine 比 C 线程多很多，但其存活时间比例小于 C线程(100%）

![goroutine_usage_time_ratio](pics/goroutine_usage_2.png)

### 并发原语使用

> 观察结论2：尽管传统的**共享内存形式的线程间通信被重度**使用，但Go程序员也使用了**大量的消息传递原语**。
>
> 隐含结论1：随着 go routines和新型并发原语的大量使用，Go程序可能引入更多并发错误。
>

**表4：各项目中使用的并发原语的总数及各自的占比统计**

- 在实际项目中，**使用共享内存相关原语还多于通道通信的并发模式**。

![go_concurrency_usage](pics/go_concurrency_usage.png)

图2：2015-2018 年各个项目的并发原语的使用比例的变化（左：共享内存，右消息传递）

![](pics/go_concurrency_usage_alongtime.png)



## Bug 研究方法

**表5：各项目中并发Bug的分类**（根据 commit 关键字搜索、过滤并分析得出 171 个并发 Bugs）

- 阻塞性 bug：goroutine 意外卡住，无法继续执行；
- 非阻塞性 bug：goroutine 执行完成，但是结果不对；

![go_bugs_maxonomy](pics/go_bugs_maxonomy.png)



## 阻塞性 Bugs

> 观察结论3：研究表明**阻塞bugs更多是由错误的消息传递导致**，而不是错误的共享内存保护，与人类普遍认知相反。

表6：**阻塞性Bugs的根因（消息传递错误58%，内存共享42%）**，考虑到共享内存的使用率比消息传递更频繁，消息传递并发Bugs率更高。

- `wait`包括 `Cond`和`WaitGroup`中的`Wait`函数；
- `Chan`表示 channel 操作，而`Chan w/`表示channel操作伴随着其它操作；
- `Lib`表示 Go 相关的消息传递的库；

![go_concurrency_blocking_bugs](pics/go_concurrency_blocking_bugs.png)

### 阻塞性Bugs根因分析

#### 共享内存的错误保护

> 观察结论4：大多数由共享内存同步引起的阻塞错误与传统语言具有相同的原因和修复方法。但仍存在一些不同，因为Go对现有原语的新实现或新的编程语义（如读写锁/WaitGroup）。
>

Mutex ：传统的问题，可由死锁检查算法通过静态程序分析检测出来；

- 单个goroutine 重复获取锁（不支持可重入），获取锁的顺序存在冲突，忘记释放锁等；

RWMutex：在Go中**写锁比读锁有更高的优先级**， 读锁的重入可能会导致死锁；

- goroutine-A 的两个RLock（无锁释放） 间执行 goroutine-B 的 Lock（写锁），导致死锁；

Wait：一般是 Cond.Wait 时，没有其它协程调用 Cond.Signal；

#### 消息传递错误

> 观察结论5：所有由消息传递引起的阻塞错误都与Go新的消息传递语义（如channel）有关。且**很难检测**，尤其与其他同步机制一起使用时。
>
> 观察结论6：与人们普遍认为的相反，**消息传递会导致比共享内存更多的阻塞错误**。注意消息传递编程中的潜在危险，并提出该领域的错误检测研究问题。

Channel：一般和通道相关的阻塞bug是因为没有向通道发送消息（或从通道接收消息）或关闭通道，而导致正在等待从通道接收消息（或等待往通道发送消息）的协程阻塞。

Channel和其它原语：一起使用造成

- 一个协程因为通道阻塞，另一个协程因为锁或wait操作阻塞。

Go中的消息库：如 Pipe，未关闭的 Pipe 的操作（如读写）时可能会被阻塞，如果使用不当也会导致并发 bugs；

### 阻塞性Bugs的修复

共享内存对应的修复bug的方法一般如下：

- 通过添加缺少的解锁操作
- 移动lock或unlock操作到合适的未知
- 移除多余的锁操作

消息传递对应的修复bug的方法一般如下：

- 添加丢失的message 或者 关闭 channel；
- 在select语句中增加default分支或在一个不同通道上的case操作；
- 将unbuffered channel替换成buffered chanel；
- channel 操作移除临界区，用共享变量替代 channel；



### 阻塞性 Bugs 检测



## 非阻塞性 Bugs



## 附：并发Bugs 示例

> 各个项目的bugs及修改commit信息见 [Github: go-concurrency-bugs](https://github.com/system-pclub/go-concurrency-bugs)