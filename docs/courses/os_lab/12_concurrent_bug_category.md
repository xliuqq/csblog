# 并发BUG分类

> 从数据库事务 (transactions, tx) 到软件和硬件支持的 [Transactional Memory](https://cs.brown.edu/~mph/HerlihyM93/herlihy93transactional.pdf)  到 [Operating System Transactions](https://www.sigops.org/s/conferences/sosp/2009/papers/porter-sosp09.pdf)，直到今天我们依然没有每个程序员都垂手可得的可靠原子性保障。

**本讲内容**：常见的并发 bugs

- 死锁
- 数据竞争
- 原子性和顺序违反

## 伤人性命的并发 Bug

在安全攸关的系统中，独立的防护模块是必不可少的。

依据调试理论，即便不是安全攸关的系统，独立于程序实现的逻辑检查 (assertions) 可以在软件造成问题之前 ***fail fast***。

### “Killed by a Machine”

Therac-25 Incident (1985-1987)

- 至少导致 6 人死亡
- 事件驱动导致的并发 bug (没有多线程)

前两者模式是合法的：第三种模式是非法的

![The Therac-25](pics/therac-25-bug.png)

在更早的产品中，使用电路互锁 (interlock) 在 assert fail 时停机

```c
assert mode in [XRay, Electron]
assert mirror in [On, Off]
assert not (mode == XRay and mirror == Off)
```

BUG发生的原因：**原子性 + 异步执行假设**导致的灾难

- 选择 X-ray Mode
  - 机器开始移动 mirror，需要 ~8s 完成

- 此时（在mirror未移动完成时），切换到 Electron Mode
  - 射线源以 X-ray Mode 运行，Mirror 尚未到位
  - 此时会触发保护机制，抛出 Malfunction 54 （该错误的解释不清晰）
    - Malfunction 54 操作说明书里面的解释是“能量已经发射，可能过高或过低”
    - 每天处理各种 Malfunction 的操作员下意识地 Continue（导致BUG模式出现）

最终解决方法

- **独立的硬件安全方案**：在大计量照射发生时直接停机

## 死锁

> A deadlock is a state in which each member of a group is waiting for another member, including itself, to take action.

### AA-Deadlock

> 不可重入锁：单个线程上锁 > 1 次；

```c
lock(&lk);
// lk = LOCKED;
lock(&lk);
// while (xchg(&lk, LOCKED) == LOCKED) ;
```

真实**系统的复杂性**等着你

- 多层函数调用
- 隐藏的控制流

### ABBA-Deadlock

哲学家吃饭问题

```c
void Tphilosopher() {
  P(&avail[lhs]);
  P(&avail[rhs]);
  // ...
  V(&avail[lhs]);
  V(&avail[rhs]);
}
```

- *T*1: P(1) - 成功, P(2) - 等待
- *T*2: P(2) - 成功, P(3) - 等待
- *T*3: P(3) - 成功, P(4) - 等待
- *T*4: P(4) - 成功, P(5) - 等待
- *T*5: P(5) - 成功, P(1) - 等待

### 死锁产生的必要条件

[System deadlocks (1971)](https://dl.acm.org/doi/10.1145/356586.356588)：死锁产生的四个必要条件

- Mutual-exclusion（互斥）：一个资源（如一张校园卡）只能被一个线程（人）拥有
- Wait-for（等待）： 一个线程（人）等其他资源（校园卡）时，不会释放已有的资源（校园卡）
- No-preemption（无抢占）：不能抢夺他人的资源（校园卡）
- Circular-chain（循环链）：形成资源（校园卡）的循环等待关系

**四个条件 “缺一不可”**

- 打破任何一个即可避免死锁
- 在程序逻辑正确的前提下 “打破” 根本没那么容易……



## 数据竞争

### 本质

> <font color='red'>不同的线程</font>同时访问<font color='red'>同一内存</font>，且<font color='red'>至少有一个是写</font>。

- 两个内存访问在 “赛跑”，“跑赢” 的操作先执行

- 例子：共享内存上实现的 Peterson 算法

<font color='red'>跑赢” 并没有想象中那么简单</font>

- Weak memory model 允许**不同观测者看到不同结果**
- Since C11: [data race is undefined behavior](https://en.cppreference.com/w/c/language/memory_model)

<img src="pics/mem-weak.png" alt="Weak memory model " style="zoom:67%;" />

### 防护

***用锁保护好共享数据***，消灭一切数据竞争

### 示例

遇到数据竞争的大部分情况

```c
// Case #1: 上错了锁
void thread1() { spin_lock(&lk1); sum++; spin_unlock(&lk1); }
void thread2() { spin_lock(&lk2); sum++; spin_unlock(&lk2); }
```

```c
// Case #2: 忘记上锁
void thread1() { spin_lock(&lk1); sum++; spin_unlock(&lk1); }
void thread2() { sum++; }
```

<font color='red'>不同的线程</font>同时访问<font color='red'>同一内存</font>，且<font color='red'>至少有一个是写</font>。

- “内存” 可以是地址空间中的任何内存
  - 可以是全部变量
  - 可以是堆区分配的变量
  - 可以是栈
- “访问” 可以是任何代码
  - 可能发生在你的代码里
  - 可以发生在框架代码里
  - 可能是一行你没有读到过的汇编代码
  - 可能时一条 ret 指令

## 原子性和顺序违反

### 并发编程的本质

人类是 sequential creature：我们只能用 sequential 的方式来理解并发

- 程序分成若干 “块”，每一块看起来都没被打断 (原子)
- 具有逻辑先后的 “块” 被正确同步
  - 例子：produce → (happens-before) → consume

并发控制的机制完全是 “后果自负” 的

- **互斥锁 (lock/unlock) 实现原子性**
  - 忘记上锁——原子性违反 (Atomicity Violation, AV)
- **条件变量/信号量 (wait/signal) 实现先后顺序同步**
  - 忘记同步——顺序违反 (Order Violation, OV)

### 那么，程序员用的对不对呢？

“Empirical study” 实证研究：收集了 105 个真实系统的并发 bugs

- MySQL (14/9), Apache (13/4), Mozilla (41/16), OpenOffice (6/2)
- [Learning from mistakes - A comprehensive study on real world concurrency bug characteristics](https://www.cs.columbia.edu/~junfeng/09fa-e6998/papers/concurrency-bugs.pdf) (ASPLOS'08, Most Influential Paper Award)

结论之一：<font color='red'>97% 的非死锁并发 bug 都是原子性或顺序错误</font>

- “人类的确是 sequential creature”

### 原子性违反 (AV)

“ABA”：Get-Then-Set，Check-Then-Use

![av-bug](pics/av-bug.png)

操作系统中还有更多的共享状态

- “TOCTTOU” - time of check to time of use

<img src="pics/tocttou.png" alt="img" style="zoom: 50%;" />

- [TOCTTOU vulnerabilities in UNIX-style file systems: An anatomical study](https://www.usenix.org/legacy/events/fast05/tech/full_papers/wei/wei.pdf) (FAST'05)

### 顺序违反 (OV)

“BA”：“怎么就没按我预想的顺序来呢？

- 例子：concurrent use after free

![img](pics/ov-bug.png)

## 课后习题/编程作业

### 1. 阅读材料

教科书 Operating Systems: Three Easy Pieces

- 第 32 章 - [Event-based Concurrency](./book_os_three_pieces/32-threads-bugs.pdf)
