# GC与内存



## 引用类型

### 强引用

普通对象引用，有强引用指向一个对象，就表明此对象还“活着”，无法被 GC。

### 软引用

> 描述一些还有用但并非必需的对象

当JVM认定**内存空间不足时才会去回收**软引用指向的对象

- 使用软引用的时候**必须检查引用是否为null**；

```java
Object obj = new Object();
SoftReference<Object> sf = new SoftReference<Object>(obj);
obj = null;
// 有时候会返回null
sf.get();　
```

### 弱引用

> 弱引用与软引用最大的区别就是弱引用比软引用的生命周期更短暂。

垃圾回收器会扫描它所管辖的内存区域的过程中，只要发现弱引用的对象，**不管内存空间是否有空闲，都会立刻回收**它。

```java
Object obj = new Object();
WeakReference<Object> wf = new WeakReference<Object>(obj);
obj = null;
//有时候会返回null
wf.get();
//返回是否被垃圾回收器标记为即将回收的垃圾
wf.isEnQueued();
```

### 幻象引用

幻象引用并不会决定对象的生命周期。即如果一个对象仅持有虚引用，就相当于没有任何引用一样，在任何时候都可能被垃圾回收器回收。

- 不能通过它访问对象

```java
Object obj = new Object();
PhantomReference<Object> pf = new PhantomReference<Object>(obj);
obj=null;
//永远返回null
pf.get();
//返回是否从内存中已经删除
pf.isEnQueued();　
```





## 内存布局

- 程序计数器
  - 当前线程所执行的字节码的行号指示器，<font color='red'>**线程私有**</font>；
- Java虚拟机栈
  - <font color='red'>**线程私有**</font>，描述Java方法执行的内存模型；
  - 为方法创建栈帧(Stack Frame)，方法所需的栈帧的局部变量空间完全确定；
  - 栈深度过深，StackOverFlowError异常；JVM可以动态扩展栈，则报OutofMemoryError异常；
- 本地方法栈
  - 为<font color='red'>**Native方法**</font>创建栈帧，StackOverFlowError和OutOfMemoryError异常；
  - 未对其使用语言、方式、数据结构强制规定，如Sun     HotSpot虚拟机直接把本地方法栈和虚拟机栈合二为一；
- 堆（heap区）：`-Xmx, -Xms`控制最大内存和初始内存
- 方法区
  - 保存虚拟机加载的类信息、常量、静态变量、即时编译器编译后代码等；
  - 垃圾回收主要针对<font color='red'>**常量池回收**</font>和对<font color='red'>**类型卸载**</font>；
- 直接内存
  - 不属于虚拟机运行时数据区，不是Java虚拟机规范中定义的内存区域；
  - `-XX:MaxDirectMemorySize`，默认与Java堆最大值一样；
  - 避免在Java堆和Native堆来回复制数据；





## GC 策略

![gc_algs.png](.pics/gc/gc_algs.png)

### 查看JDK默认的GC策略

```bash
$ java -XX:+PrintCommandLineFlags -version
```



### Serial（单线程）

复制算法

- JDK1.3.1之前的新生代收集器；

- 单线程收集器，Stop-the-world；

- Client 模式新生代默认收集器；

### ParNew(并行回收)

> -XX:UseParNewGC

Serial 的多线程版

- Server 模式下的 **新生代的默认垃圾收集器**



### Parallel Scavenge(吞吐量优先，JDK 8默认)

> -XX:+UseParallelGC

复制算法：

- 关注<font color='red'>**吞吐量**</font>=运行代码时间 /（运行代码时间 +垃圾收集时间），后台应用；
- `-XX:MaxGCPauseMills，-XX:GCTimeRatio`设置GC时间和吞吐量；
- 自适应调节策略：`-XX:+UseAdaptiveSizePolicy`



### Serial Old

标记-整理算法

- 单线程收集器，Client模式使用；

- Server模式，1）JDK1.5之前与Parallel Scavenge使用；2）作为CMS后备预案，并发收集器失败时使用；



### Parallel Old

标记-整理算法

- 多线程，JDK1.6之后（来源于Parallel Scavenge对老年代并行回收的需要）；

- 注重吞吐量和CPU资源敏感，使用Parallel Scavenge + Parallel Old；



### Concurrent-Mark-Sweep(低延迟，JDK9废弃)

> CMS 是一种高度可配置的复杂算法，因此给 JDK 中的 GC代码库带来了很多复杂性。

标记-清除算法

- 最短回收停顿时间，交互式应用；

- **初始标记 -> 并发标记 -> 重新标记 -> 并发清除**：

- - 初始标记和重新标记需要Stop-The-World；
  - 初始标记：标记GC Roots能直接关联的对象；单线程
  - 并发标记：进行GC Roots Tracing的过程；多线程
  - 重新标记：修正并发标记期间因用户程序继续运行造成标记变动的对象的标记记录；

缺点：

- 对CPU资源敏感；
- 无法处理浮动垃圾（CMS运行期间老年代预留内存不够，导致Seria Old进行老年代收集）；
- 标记-清除算法带来碎片，`-XX:+UseCMSCompactAtFullCollection` 在CMS要进行Full GC时开启内存碎片整理；



### Garbage First（G1，JDK 9默认）

> 和之前的各类回收器不同，它同时`兼顾年轻代和老年代`。对比其他回收器，或者工作在年轻代，或者工作在老年代。

`G1依然属于分代型垃圾回收器`，它会区分年轻代和老年代，年轻代依然有Eden区和Survivor区。但从堆的结构上看，它不要求整个Eden区、年轻代或者老年代都是连续的，也不再坚持固定大小和固定数量。

![gc_g1_flow.jpg](.pics/gc/gc_g1_flow.jpg)

- 并行与并发、分代收收集、空间整合（无碎片）、可预测的停顿（时间模型）；
- 将堆划分为大小相等的多个 Region，每次回收价值最大的Region（回收空间大小以及所需时间经验值）；
- 每个Region有对应的Remembered     Set（避免全堆扫描），当程序对引用类型数据写操作时，产生Write Barrier中断写操作，检查引用对象是否处于不同Region（分代中就是检查是否老年代对象引用新生代对象），是则有CardTable将引用信息加入被引用对象所属Region的Remembered     Set中。GC根节点枚举时加入Remembered Set避免全堆扫描；



### ZGC(低延迟GC)



#### JVM的GC线程数的计算





## GC 算法

### 引用计数法

循环引用问题：可达性分析：

- 从GC Roots沿着引用链，判断对象可达与否；

- GC Roots ：
  - 虚拟机栈（栈帧中的本地变量表）中引用的对象；
  - 方法区中类静态属性引用的对象；
  - 方法区中常量引用的对象；
  - 本地方法栈中JNI（Native方法）引用的对象；

### 标记-清理算法

标记回收的对象，清理这些对象；

- 标记、清理效率不高；产生内存碎片；

### 复制算法

> 适用于新生代，频繁申请和释放

内存分成相等两块，当一块内存使用完时，将存活的对象复制到另一块内存上，并清理之前一块内存；

- 内存分成一块较大的 Eden 空间和两块较小的 Survivor 空间
- 回收时，将 Eden 和 Survivor 中存活的对象一次性复制到另一个Survivor空间上，清理之前的Eden和Survivor空间；
- 当 Survivor 空间不够时，需要依靠老年代进行分配担保，即多余的存活对象放入老年代中；

### 标记-整理算法

标记回收对象，存活的对象**向一端移**，清除边界以外内存，消除内存碎片；



### 内存回收算法实现

枚举根节点

- Stop The World

  - 准确式GC，虚拟机直接得知哪些地方存放着对象引用，通过OopMap数据结构，JIT编译过程会在特定位置记录栈和寄存器中哪些位置有引用；

- SafePoint：程序执行时并非在所有地方都能停顿下来开始GC，只有到达安全点时才能暂停；

  - 安全点的选取以程序“是否有让程序长时间执行的特征”即指令序列复用，如方法调用、跳转等；

  - 抢占式中断：GC时，中断所有线程，若线程中断地方不是安全点，则恢复线程跑到安全点，很少使用；

  - 主动式中断：GC需要中断线程时，仅设置标志，各线程执行时主动轮询该标志（到达安全点时或创建对象需要分配内存时轮询），发现中断标志为真时就自己中断挂起；

- 安全区域：当程序“不执行”（Sleep或Blocked状态）时，线程如何进入GC？

  - 安全区域指在一段代码中，引用关系不会发生变化，在这个区域中的任意地方开始GC都安全；
  - 将线程Sleep这些加入Safe Region中，线程进入Safe Region代码时做标识，在此期间JVM发起GC时，不需管自己为Safe Region状态的线程；线程离开Safe Region时，检查系统是否已经完成了根节点枚举（或整个GC过程），完成则线程继续执行，否则等待知道收到可以安全离开Safe       Region的信号；