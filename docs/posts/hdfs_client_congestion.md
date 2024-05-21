---
date: 2023-12-18
readtime: 5
categories:
  - HDFS
---

# HDFS 读写限速

当磁盘满负荷时，希望能够降低读写的速率，避免 HDFS 进程卡住，整个HDFS 不可用，导致 Client Socket 异常，作业失败。

当前（2023.12.18，HDFS 3.4 版本）：

- 对于读入速率：并没有搜到 HDFS 相关可以对读取速率做控制；
- 对于写入速率：可以通过`dfs.client.congestion.backoff.mean.time`、`dfs.client.congestion.backoff.max.time`控制写入拥塞时 Client 的等待时间，用`dfs.pipeline.congestion.ratio` 来控制 DataNode 被判断阻塞时的跟CPU核数的比率。



<!-- more -->



## 写入速率

> 相关 JIRA ISSUE
>
> - [HDFS-7270: Add congestion signaling capability to DataNode write protocol](https://issues.apache.org/jira/browse/HDFS-7270)
>
> - [HDFS-8008: Support client-side back off when the datanodes are congested](https://issues.apache.org/jira/browse/HDFS-8008)
> - [HDFS-8009: Signal congestion on the DataNode](https://issues.apache.org/jira/browse/HDFS-8009)
> - [HDFS-17171: CONGESTION_RATIO should be configurable](https://issues.apache.org/jira/browse/HDFS-17171)
> - [HDFS-17242: Make congestion backoff time configurable](https://issues.apache.org/jira/browse/HDFS-17242)

当 DataNode 被认为阻塞时，Client 会进行 backoff 等待一段时间后，再进行写入。

DataStreamer （负责写）中 `backOffIfNecessary` 函数根据 Write Pipeline Ack 中的 ECN 表示来进行 Sleep

- ECN 的设置：DataNode 中的 `getECN()`，DataNode 有标志位可以表示 CONGESTED（拥塞）

```java
// see https://issues.apache.org/jira/browse/HDFS-8009
public PipelineAck.ECN getECN() {
    if (!pipelineSupportECN) {
        return PipelineAck.ECN.DISABLED;
    }
    double load = ManagementFactory.getOperatingSystemMXBean().getSystemLoadAverage();
    // 系统平均负载 > 1.5 * 核数
    return load > NUM_CORES * CONGESTION_RATIO ? PipelineAck.ECN.CONGESTED :
    PipelineAck.ECN.SUPPORTED;
}
```

### 拥塞的判断

`系统平均负载 > 1.5 * 核数`(Hadoop 3.4 版本可配置)，注意 $负载 \ne 利用率 $

> The v0 patch allows the datanode to signal congestion when the load of the system is **over 1.5 times of the number of available cores**, which means that there are **at least 50% more processes are waiting to be scheduled** in the OS. We found it an effective metric to signal that the DNs are undergoing massive amount of writes.

为什么**磁盘慢时（大量磁盘使用时），CPU 负载会飙高**。

- CPU负载显示的是在一段时间内<font color='red'>**CPU正在处理以及等待CPU处理的进程数之和的统计信息**</font>（waiting threads and tasks along with processes being executed），也就是CPU使用队列的长度的统计信息。

LInux 系统中还CPU负载包含处于**不可中断状态的任务**（`TASK_UNINTERRUPTIBLE` 或 `nr_uninterruptible`），被认为是系统负载：

- 状态被标志为“D”：`uninterruptible sleep`

- 通常**等待IO设备、等待网络**的时候，进程会处于`uninterruptible sleep`状态；