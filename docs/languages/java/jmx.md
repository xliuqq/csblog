# JMX

**JMX(Java Management Extensions)是一个为应用程序植入管理功能的框架**。JMX是一套标准的代理和服务，主要用于对JAVA应用程序和JVM进行监控和管理。

JConsole 和 JVisualVM 中能够监控到JAVA应用程序和JVM的相关信息都是通过JMX实现的。

<img src=".pics/jmx/jmx_arch.png" alt="img" style="zoom:80%;" />

## 连接

### MBean本地连接

**jps，jinfo，jmap，jstat**等jdk自带的命令去查询进程的状态，这其中的原理就是，当java进程启动后，会创建一个用于本机连接的“**localConnectorAddress**”放到当前用户目录下，当使用jps等连接时，会到当前用户目录下取到“localConnectorAddress”这个JMX连接地址并连接。

### MBean远程连接

若想远程连接访问，需要把mbean注册到**platformMBeanServer**里面，并在启动进程时加**jmx参数**指定，可以通过jconsole，jvisualvm远程访问。

```shell
-Dcom.sun.management.jmxremote=true                   相关 JMX 代理侦听开关
-Djava.rmi.server.hostname=192.168.1.2                服务器端的IP
-Dcom.sun.management.jmxremote.port=29094             相关 JMX 代理侦听请求的端口
-Dcom.sun.management.jmxremote.ssl=false              指定是否使用 SSL 通讯
-Dcom.sun.management.jmxremote.authenticate=false     指定是否需要密码验证
```

## MXBean

JDK提供了一些JVM检测的API，**java.lang.management** 包，提供了许多**MXBean**的接口类，可以很方便的获取到JVM的**内存、GC、线程、锁、class、甚至操作系统层面**的各种信息。



### OperatingSystemMXBean

```java
import com.sun.management.OperatingSystemMXBean;

OperatingSystemMXBean osBean = ManagementFactory.getPlatformMXBean(OperatingSystemMXBean.class);

osBean.getProcessCpuLoad();  // 进程的CPU负载
osBean.getSystemCpuLoad();   // 系统的CPU负载
```



### MemoryMXBean

可以获取当前Heap和NonHeap的内存使用状况（初始值，使用值，提交值，最大值）。



### MemoryManagerMXBean

内存管理器的管理接口。内存管理器管理 Java 虚拟机的一个或多个内存池。Java 虚拟机具有一个或多个内存管理器。

- getMemoryPoolNames() ：返回此内存管理器管理的内存池名称
- getName() ：返回表示此内存管理器的名称

看出当前应用使用的垃圾回收算法及其应用的堆内存分区。



### MemoryPoolMXBean

获取内存池的当前使用量（估计值）和峰值使用量。

- getName()：内存池的名称，如Code Cache, Metaspace, PS Eden Space等；
- getType()：堆内还是堆外内存；
- getUsage() 和 getPeakUsage()



### GarbageCollectorMXBean

可以获取GC的次数和时间。



### BufferPoolMXBean

获取缓冲池的容量和已使用量。



### ThreadMXBean



- `ThreadInfo[] dumpAllThreads(boolean lockedMonitors, boolean lockedSynchronizers)`