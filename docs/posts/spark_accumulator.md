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



## Transform 中的累加器重算

由上一节可知：即使一个分区中调用多次累加器的`add`函数，也只会在 Task 完成后由 Driver更新。

- 因此一个分区只要触发一次`add`即可。

Driver调用累加器的`merge`函数时，如何知道重算（是个已经算过的分区）？

- 用 map 保存`(分区ID，分区统计量)`，当发现分区ID已经存在时，不会更新；
- 进一步，分区的重算是由 Stage的重算产生的，那么之前的 Stage 的所有分区肯定都已经计算过
  - 一种是 Shuffle 数据无法找到，导致的重算，此时之前的Stage已经计算过
  - 一种是 RDD 多次使用导致的 Stage 重算，此时，之前的Stage的分区已经计算
- 因此，只需要统计当前分区的总数即可，当分区总数超过时，表明触发了重算，无需累加。
  - 失败的 Task 不会发送累加器的数据（见上一节）。



具体的示例，可以见[示例：Spark串接流统计输出数据行数](https://gitee.com/oscsc/bigdatatech/blob/master/spark/module-progress/README.md)