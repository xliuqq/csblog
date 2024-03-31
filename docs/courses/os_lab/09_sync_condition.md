# 并发控制：同步（1）

**本讲内容**：并发控制：同步

- 同步问题的定义
- 生产者-消费者问题
- 条件变量

## 同步问题

### 同步 (Synchronization)

两个或两个以上随**时间变化的量**在变化过程中**保持一定的相对关系**

- 同步电路 (一个时钟控制所有触发器) 
- iPhone/iCloud 同步 (手机 vs 电脑 vs 云端)
- 变速箱同步器 (合并快慢速齿轮) 
- 同步电机 (转子与磁场转速一致) 
- 同步电路 (所有触发器在边沿同时触发)

异步 (Asynchronous) = 不需要同步

- 上述很多例子都有异步版本 (异步电机、异步电路、异步线程)

### 并发程序中的同步

并发程序的步调很难保持 “完全一致”

- 线程同步：<font color='red'>在某个时间点共同达到互相已知的状态</font>

### 生产者-消费者问题

> 99% 的实际并发问题都可以用生产者-消费者解决。

并行计算基础：计算图

- 计算任务构成有向无环图
  - (*u*,*v*)∈*E* 表示 *v* 要用到前 *u* 的值
- 只要调度器 (生产者) 分配任务效率够高，算法就能并行
  - 生产者把任务放入队列中
  - 消费者 (workers) 从队列中取出任务

**示例：括号打印**

```c
void Tproduce() { while (1) printf("("); }
void Tconsume() { while (1) printf(")"); }
```

在 `printf` 前后增加代码，使得打印的括号序列满足

- 一定是某个合法括号序列的前缀
- 括号嵌套的深度不超过
  - *n*=3, `((())())(((` 合法
  - *n*=3, `(((())))(()))` 不合法

### 生产者-消费者问题：实现

用互斥锁实现括号问题：

- 左括号：嵌套深度 (队列) 不足 n* 时才能打印
- 右括号：嵌套深度 (队列) >1 时才能打印
- <font color='red'>当然是等到满足条件时再打印了(代码演示)</font>
  - <font color='red'>用互斥锁保持条件成立</font>：会出现无用的申请锁->释放，CPU资源浪费

测试用例和检查工具，评估并发程序的正确性。

## 条件变量

### 同步问题：分析

> 线程同步由<font color='red'>条件不成立等待</font>和<font color='red'>同步条件达成继续</font>构成。

生产者/消费者问题

- Tproduce 同步条件：`CAN_PRODUCE (count < n)`
- Tproduce 达成同步：Tconsume `count--`
- Tconsume 同步条件：`CAN_CONSUME (count > 0)`
- Tconsume 达成同步：Tproduce `count++`

### 理想中的同步 API

```c
wait_until(CAN_PRODUCE) {
  count++;
  printf("(");
}

wait_until(CAN_CONSUME) {
  count--;
  printf(")");
}
```

若干实现上的难题

- 正确性：大括号内代码执行时，其他线程不得破坏等待的条件
- 性能
  - 不能 spin check 条件达成
  - 已经在等待的线程怎么知道条件被满足？

### 条件变量：理想与实现之间的折衷

一把互斥锁 + 一个 “条件变量” + 手工唤醒

- `wait(cv, mutex)`
  - 调用时必须**保证已经获得 mutex**
  - **wait 释放 mutex、进入睡眠状态**
  - 被唤醒后需要**重新执行 lock(mutex)**
- `signal/notify(cv)`：随机私信一个等待者：醒醒
  - 如果有线程正在等待 cv，则唤醒其中一个线程
- `broadcast/notifyAll(cv) `：  叫醒所有人
  - 唤醒全部正在等待 cv 的线程

### 条件变量：实现生产者-消费者

代码演示 & 压力测试 & 模型检验

- 错误的示例（下面的代码）：
  - 条件判等的时候，需要用 while（防止代码的错误唤醒或者os的假唤醒）；
  - 需要两个条件变量，Tproduce 只希望唤醒消费者；Tconsume 只希望唤醒生产者，防止无效唤醒；

```c
void Tproduce() {
  mutex_lock(&lk);
  // 需要用 while 判断，并且用两个条件变量避免无效的唤醒
  if (!CAN_PRODUCE) cond_wait(&cv, &lk);
  printf("("); count++; cond_signal(&cv);
  mutex_unlock(&lk);
}

void Tconsume() {
  mutex_lock(&lk);
  // 需要用 while 判断，并且用两个条件变量避免无效的唤醒
  if (!CAN_CONSUME) cond_wait(&cv, &lk);
  printf(")"); count--; cond_signal(&cv);
  mutex_unlock(&lk);
}
```

### 条件变量：正确的打开方式

同步的本质：`wait_until(COND) { ... }`，因此：

- <font color='red'>需要等待条件满足时</font>

```c
mutex_lock(&mutex);
// while 循环，被唤醒时需要判断条件是否成立
while (!COND) {
  wait(&cv, &mutex);
}
assert(cond);  // 互斥锁保证条件成立
mutex_unlock(&mutex);
```

- <font color='red'>任何改动使其他线可能被满足时</font>

```c
mutex_lock(&mutex);
// 任何可能使条件满足的代码（防止唤醒不能执行的线程，导致死锁）
broadcast(&cv);
mutex_unlock(&mutex);
```



## 条件变量：应用

### 万能并行计算框架 (M2)

```c
struct work {
  void (*run)(void *arg);
  void *arg;
}

void Tworker() {
  while (1) {
    struct work *work;
    wait_until(has_new_work() || all_done) {
      work = get_work();
    }
    if (!work) break;
    else {
      work->run(work->arg); // 允许生成新的 work (注意互斥)
      release(work);  // 注意回收 work 分配的资源
    }
  }
}
```



### 条件变量：更古怪的习题/面试题

有三种线程

- Ta 若干: 死循环打印 `<`
- Tb 若干: 死循环打印 `>`
- Tc 若干: 死循环打印 `_`

任务：

- 对这些线程进行同步，使得屏幕打印出 `<><_` 和 `><>_` 的组合

使用条件变量，只要回答三个问题：

- 打印 “`<`” 的条件？`A,C,E`
- 打印 “`>`” 的条件？`A,B,F`
- 打印 “`_`” 的条件？`D`

**状态机**

 ```mermaid
 graph LR
 	A --"<"--> B
 	B --">"--> C
 	C --"<"--> D
 	A --">"--> E
 	E --"<"--> F
 	F --">"--> D
 	D --"_"--> A
 	
 ```



## 课后习题/编程作业

### 1. 阅读材料

教科书 Operating Systems: Three Easy Pieces:

- [第 29 章 - Locked Data Structures](./book_os_three_pieces/29-threads-locks-usage.pdf)
- [第30 章 - Condition Variables](./book_os_three_pieces/30-threads-cv.pdf)

### 2. 编程实践

运行示例代码并观察执行结果。建议大家先不参照参考书，通过两个信号量分别代表 Tproduce 和 Tconsume 的唤醒条件实现同步。

### 3. 实验作业

开始 [M2](./M2_plcs.md) 和 [L1](./L1_pmm.md) 实验作业。