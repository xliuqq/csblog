---
date: 2024-11-19
readtime: 20
categories:
  - spark

---



# Spark Accumlator 解析

问题：

- Accumlator 的更新粒度是 Task 级别，还是什么粒度？
- Task 失败时的累加器信息，是否仍会更新，出现重复的情况？
- Action 中的累加器如何保证只执行一次？
- Transform 中的累加器是否有办法解决重算时多次更新的问题？



<!-- more -->



## 累加器的更新流程

整体流程大致如下:

- Driver 端序列化时：闭包传递的是空的对象（`AccumulatorV2::writeReplace`）；
- Executor 端反序列化时：通过 TaskContext 进行注册（`AccumulatorV2::readObject`）；
- Executor 端执行 Task 时：反序列化得到的AccumulatorV2进行正常更新（`add`）；
- Executor 端Task 完成时向 Driver 发送 DirectTaskResult，包含本Task的计数器的信息；
  - 如果累加器的`countFailedValues`设置为false，则Task异常/失败时不会携带该累加器。
- Driver 端在处理`CompletionEvent`（ Task 完成时）时，获取Task的累加器的信息并更新到Driver；

Driver 端的`CompletionEvent`的处理逻辑`handleTaskCompletion`，对于累加器的更新流程：

- Task 成功
  - `ResultTask`：如果 Task 的 outputId 未标记完成，则执行累加器更新；（保证每个Task仅执行一次）；
  - 其它（`ShuffleMapTask`），更新累加器
- Task 异常失败/被杀死：更新累加器

SparkContext 中注册累加器，不暴露 AccumulatorV2 的`countFailedValues`字段，因此都为 false。

- **更新的粒度是 Task 级别**，即使Task内部多次调用`add`，也只会在完成时整体更新；
- 对于**失败的 Task，其累加器不会重复更新**（失败的Task不会发送其累加器信息)；
- Action 是根据 **ResultTask的 `outputId`保证每个Task仅执行一次**
  - [a speculative task or a resubmitted task](https://github.com/apache/spark/pull/19877/files) 会多次执行，resubmit 的 task 不会影响（见第二点）。



## 累加器的相关线程

> Accumulator 不是线程安全的：[[SPARK-21425\] LongAccumulator, DoubleAccumulator not threadsafe - ASF JIRA](https://issues.apache.org/jira/browse/SPARK-21425)
>
> - 原生的LongAccumulator，其内部的变量，并[没有用 volatile 修饰或者使用Atomic 变量 ](https://github.com/apache/spark/pull/15065)。

Executor 端：

- 写线程：运行任务的线程。
- 读线程：运行任务的线程和执行器心跳线程。
  - 当报告心跳时，executor 将收集 Accumulator中的所有当前值并报告给驱动程序。

Driver端：

- 写线程：
  - 任务处理线程：`DAGSchedulerEventProcessLoop`，`handleTaskCompletion`负责累加器`merge`
- 读线程：
  - 心跳处理线程：只在UI上使用它们，用户代码看不到它们。如果没有同步，最坏的情况是用户无法在UI上看到最新的值，但这是可以接受的。
  - 任务处理线程：`handleTaskCompletion`会读取`value`，将看到所有最新和正确的值。

因此，如果要在Driver进程中的其它线程不能直接读取`value`的值，因为一读一写线程，

- 采用`volatile`内存屏障修饰相关累加量或采用线程安全的形式。
- 或通过 `SparkListener`监听`TaskEnd`事件中的`Accumulator`相关的信息。



## Transform 中的累加器重算

> 实验代码见：[spark/module-progress](https://gitee.com/oscsc/bigdatatech/tree/master/spark/module-progress)

由上一节可知：即使一个分区中调用多次累加器的`add`函数，也只会在 Task 完成后由 Driver更新。

- 因此一个分区只要触发一次`add`即可。

Driver调用累加器的`merge`函数时，如何知道重算（是个已经算过的分区）？

- 用 map 保存`(分区ID，分区统计量)`，当发现分区ID已经存在时，不会更新；
- 进一步，分区的重算是由 Stage的重算产生的，那么之前的 Stage 的所有分区肯定都已经计算过
  - 一种是 Shuffle 数据无法找到，导致的重算，此时之前的Stage已经计算过
  - 一种是 RDD 多次使用导致的 Stage 重算，此时，之前的Stage的分区已经计算
- 因此，只需要统计当前分区的总数即可，当分区总数超过时，表明触发了重算，无需累加。
  - 失败的 Task 不会发送累加器的数据（见上一节）。



但是，<font color='red'>**如果 RDD 先执行部分分区的计算，再执行整体的计算，则无法得到正确的值**</font>。

- 如下面所示的代码，假设对于`newRdd `先执行了部分分区的操作，再执行所有分区数据操作；
- 依据分区总数，就会计算前面的分区两次，导致结果出错；
- 因此，使用scala原生的`BitSet`，使用位图索引，避免分区计算多次，同时节省内存。

```scala
// val rdd: RDD[(K, V)]

val newRdd: RDD[(Int, Array[Int])] = rdd.mapPartitions(iter => {
    val id = TaskContext.getPartitionId()
    new Iterator[(Int, Array[Int])] {
      private var count = 0

      override def hasNext: Boolean = {
        val flag = iter.hasNext
        if (!flag) {
          accum.add(id, count) // 每个分区只在完成时发送，因此只需 add 一次
        }
        flag
      }
      override def next(): (Int, Array[Int]) = {
        count += 1
        iter.next()
      }
    }
})

newRDD.take(10)

newRDD.foreach()
```





具体的示例，可以见[示例：Spark串接流统计输出数据行数](https://gitee.com/oscsc/bigdatatech/blob/master/spark/module-progress/README.md)
