# Java版本特性（TODO）

| 版本      | 开始日期  | 结束日期  | 延期结束日期 |
| --------- | --------- | --------- | ------------ |
| 8（LTS）  | 2014年3月 | 2022年3月 | 2030年12月   |
| 11（LTS） | 2018年9月 | 2023年9月 | 2026年9月    |
| 17（LTS） | 2021年9月 | 2026年9月 | 2029年9月    |
| 21（LTS） | 2023年9月 | 2028年9月 | 2031年9月    |

**Open JDK发行版可选**

Open JDK 虽然没有官方的LTS版本，但是开源社区有支持。会有一些公司或组织基于Open JDK做发行版，提供LTS，**建议AdoptOpenJDK。**。

| 名称                        | 支持团队                                              | 官网                                                         |
| --------------------------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| AdoptOpenJDK                | Amazon，Microsoft，IBM，Red Hat，Pivotal(EMC和VMware) | [AdoptOpenJDK - 开源，预建OpenJDK二进制文件](https://adoptopenjdk.net/) |
| Alibaba Dragonwell （龙井） | 阿里巴巴                                              | [Dragonwell (dragonwell-jdk.io)](https://dragonwell-jdk.io/) |
| Tencent Kona                | 腾讯                                                  | [Home · Tencent/TencentKona-8 Wiki · GitHub](https://github.com/Tencent/TencentKona-8/wiki) |
| Microsoft JDK               | Microsoft微软                                         | [Microsoft Build of OpenJDK](https://www.microsoft.com/openjdk) |
| 毕昇JDK                     | 华为                                                  | [毕昇JDK-鲲鹏社区 (hikunpeng.com)](https://www.hikunpeng.com/developer/devkit/compiler/jdk) |
| Amazon Corretto             | Amazon亚马逊                                          | [Amazon Corretto](https://aws.amazon.com/cn/corretto/)       |

## Java 17

第一个支持ZGC的LTS版本。

1. 试验ZGC，需要在JVM配置（jdk11）
2. 文本块升级。（jdk13）
3. switch支持lambda（jdk13预览，jdk14）
4. ZGC 可用于生产环境（jdk15）
5. record（jdk14,15预览，jdk16）
6. Realed class密封类（jdk15,16预览，jdk17）
7. 统一日志支持异步日志刷新（jdk17）



## Java 11

### 模块化

module，open，exports；   http://www.cnblogs.com/IcanFixIt/p/7144366.html 





### jshell



### 反应式流 （ Reactive Streams ）

`java.util.concurrent.Flow` 类，Flow.Publisher、Flow.Subscriber、Flow.Subscription 和 Flow.Processor 等 4 个核心接



### 变量句柄

`VarHandle` 类，变量句柄是一个变量或一组变量的引用，包括静态域，非静态域，数组元素和堆外数据结构中的组成部分等

- - 取代 `java.util.concurrent.atomic` 包以及 `sun.misc.Unsafe` 类的功能；



### MethodHandles 

改进方法句柄，添加更多静态方法创建不同类型方法句柄

- arrayConstructor：创建指定类型的数组。

- arrayLength：获取指定类型的数组的大小。
- varHandleInvoker 和 varHandleExactInvoker：调用 VarHandle 中的访问模式方法。
- zero：返回一个类型的默认值。
- empty：返 回 MethodType 的返回值类型的默认值。
- loop、countedLoop、iteratedLoop、whileLoop 和 doWhileLoop：创建不同类型的循环，包括 for 循环、while 循环 和 do-while 循环。
- tryFinally：把对方法句柄的调用封装在 try-finally 语句中。



### 局部变量类型推断

```java
var list = new ArrayList<String>(); 
```



### 应用程序类数据共享

Class Data Sharing特性在原来的 bootstrap 类基础之上，扩展加入了应用类的 CDS (Application Class-Data Sharing) 支持；当多个 Java 虚拟机（JVM）共享相同的归档文件时，还可以减少动态内存的占用量，同时减少多个虚拟机在同一个物理或虚拟的机器上运行时的资源占用。



### 进程

#### Process API

> 非常适合耗时较长的进程调用。

`ProcessHandle` 提供了对**本地进程的控制**，可以监控其存活，查找其子进程，查看其信息，甚至销毁它。

#### Stack-Walking API

一个标准API用于访问当前线程栈。



#### 线程-局部管控

将允许**在不运行全局 JVM 安全点的情况下实现线程回调**，由线程本身或者 JVM 线程来执行，同时保持线程处于阻塞状态，这种方式**使得停止单个线程变成可能**，而不是只能启用或停止所有线程。



### JNI Native Header

合并javac和java命令（jdk11）

- 生成 Native-Header 直接用 java 命令；

- 简化启动单个源代码文件，java     HelloWorld.java；



### 异步非阻塞的Http Client

- 支持异步非阻塞的Http Client，支持HTTP/1.1和HTTP/2；



### GC

**默认 G1 垃圾回收器**：废弃CMS

**并行全垃圾回收器 G1**：之前 Java 版本中的 G1 垃圾回收器执行 GC 时采用的是基于单线程标记扫描压缩算法（mark-sweep-compact），采用并行化 mark-sweep-compact 算法，并使用与年轻代回收的相同数量的线程（-XX：ParallelGCThread）；

**实验性质**：

- Epsilon：低开销垃圾回收器，-XX:+UseEpsilonGC；

- ZGC：可伸缩低延迟垃圾收集器，--with-jvm-features=zgc；



### incurator  HTTP/2？

支持异步非阻塞的Http Client，支持HTTP/1.1和HTTP/2；



### FlightRecording

飞行记录器（JDK8时为商业版）：低开销的事件信息收集框架，主要用于对应用程序和 JVM 进行故障检查、分析，-XX:StartFlightRecording；