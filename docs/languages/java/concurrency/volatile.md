# Volatile

> **可见性**：内存同步障
>
> - 当**写**一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值立即刷新回主内存中。
> - 当**读**一个volatile变量时，JMM会把该线程对应的本地内存设置为无效，重新回到主内存中读取最新共享变量。
>
> **有序性**：禁重排！



**一写多读场景，可以不用锁，用volatile修饰即可。**

- **引用变量的赋值是原子性操作**

## 内存屏障

> **通过插入内存屏障禁止在内存屏障前后的指令执行重排序优化**。

<img src=".pics/volatile/store_load.png" alt="image-20231023163748685" style="zoom: 80%;" />

## 双重校验

指令重排与双重校验单例，通过volatile屏蔽指令重排

```java
public class Singleton {
    private static volatile Singleton instance;

    public static Singleton getSingleton() {
        if (instance == null) { // Single Checked
            synchronized (Singleton.class) {
                if (instance == null) { // Double Checked
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

>《The Java Language Specification, Java SE 7 Edition》（后文简称为 java 语言规范）
>
>- 所有线程在执行 java 程序时必须要遵守 intra-thread semantics。intra-thread semantics 保证**重排序不会改变单线程内的程序执行结果**

如果 instance 不用 volatile 修饰，则可能会出现问题：

正常的new操作应该是：

1）分配一块内存；

2）在内存上完成Singleton对象的初始化；

3）把内存地址给instance；

但实际上被优化后可能是这样的：

1）分配一块内存；

2）将内存地址给instance；

3）在内存上完成Singleton对象的初始化；

我们假设有两个线程A、B，A执行getIntance到指令2时，线程切换到线程B，线程B看到instance已经不为空，认为可用了。

但这时还没有进行对象的初始化，这时使用instance，就可能出现问题。

 ![img](pics/volatile_sample.png)