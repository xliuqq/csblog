# 同步&互斥

共享资源的互斥访问

共享资源同步（同步是要保证两个线程事件执行的时序关系）

## synchronized

```java
synchronized {
}
```

## wait & notify

```java
synchronized {
    while (condition is not matched) {
        obj.wait();  // (Release lock, and reacquired on wakeup)
    }
}

synchronized {
    obj.notify();
}
```

- **wait 方法会释放锁**，进行对象的等待池，而 **Thread的sleep，yield方法不会释放锁**；
- **notify()会在同步方法块执行完后，再释放锁**；
- 在没有通知的情况下，等待线程也可能（但很少）会苏醒过来，被称为“伪唤醒”（spurious wakeup）。
- 一般情况下，优先使用notifyAll，而不是notify。

## CAS

> CAS : Compare And Swap，乐观锁机制

Java的原子类AtomicXXX 和 XXXAdder 都是使用CAS保证了线程安全。

- **ABA问题**：我们拿到期望值A，再用CAS写新值C，这两个操作并不能保证原子性。因此，就有可能出现实际值已经改为B，又改回A，这种情况。当执行CAS时，不会识别ABA问题，只要看实际值还是A，就执行了。
  - 通过版本号，即一个递增的值，可以解决 ABA 的问题

- 原子化的对象引用：对象引用都是封装类型，支持CAS操作。相关实现包括：AtomicReference、AtomicStampedReference和AtiomicMarkableReference。AtomicStampedReference和AtiomicMarkableReference解决了ABA问题。
- 原子化数组：AtomicIntegerArray、AtomicLongArray和AtomicReferenceArray；
- 原子化对象属性更新器：AtomicIntegerFieldUpdater、AtomicLongFiledUpdater和AtomicReferenceFieldUpdater
- 原子化的累加器：DoubleAccumulator、DoubleAdder、LongAccumulator 和 LongAdder



## VarHandle

JDK8，如果要**原子性地增加某个字段的值**，到目前为止我们可以使用下面三种方式：

- 使用**AtomicInteger**来达到这种效果，这种间接管理方式增加了空间开销，还会导致额外的并发问题；

- 使用原子性的**FieldUpdaters**，利用了反射机制，操作开销也会更大；

- 使用sun.misc.Unsafe提供的JVM内置函数API，虽然这种方式比较快，但它会损害安全性和可移植性。

在 VarHandle 出现之前，这些潜在的问题会随着原子API的不断扩大而越来越遭。

JDK 9 新增定义。

**变量句柄：**

- 是一个变量或一组变量的引用，包括静态域，非静态域，数组元素和堆外数据结构中的组成部分等

VarHandle 的出现替代了 java.util.concurrent.atomic 和 sun.misc.Unsafe 的部分操作。并且提供了一系列标准的内存屏障操作，用于更加细粒度的控制内存排序。在安全性、可用性、性能上都要优于现有的API。

VarHandle 可以与任何字段、数组元素或静态变量关联，支持在不同访问模型下对这些类型变量的访问，包括简单的 read/write 访问，volatile 类型的 read/write 访问，和 CAS(compare-and-swap)等。



## CountDownLatch 

**线程间同步，多用于任务划分场景，标记任务是否完成；**

```java
// ExecutorService用于计时结束
CountDownLatch latch = new CountDownLatch(1);
latch.countDown();
latch.wait();

// 线程间同步
class Driver { // ...
    void main() throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(N);

        for (int i = 0; i < N; ++i) // create and start threads
            new Thread(new Worker(startSignal, doneSignal)).start();

        doSomethingElse();            // don't let run yet
        startSignal.countDown();      // let all threads proceed
        doSomethingElse();
        doneSignal.await();           // wait for all to finish
    }
}

class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;
    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }
    public void run() {
        try {
            
            
            startSignal.await();
            doWork();
            doneSignal.countDown();
        } catch (InterruptedException ex) {} // return;
    }

    void doWork() { ... }
}
```



## 静态初始化死锁

### 示例1

**类A的静态初始化涉及到多个线程，且多个线程中又会调用类A的静态初始化或静态方法；**

(示例1：**lambda方法被编译成类的静态方法**，多线程执行静态方法会等待类初始化完成，导致死锁）

```java
import java.util.stream.IntStream;

public class StaticInitialDeadLockDemo {
    static {
        long count = IntStream.range(0, 10000).parallel().map(i -> i).count();
        System.out.println("done:" + count);
    }

    public static void main(final String[] args) {}
}

```

### 示例2

类A的静态初始化和类B的**静态初始化互相引用**，不同的线程中同时调用类A和类B的初始化；（示例2）

```java
public class StaticInitialDeadLockDemo2 {
    public static void main(String[] args) throws Exception {
        System.out.println("Main begin...");
        new Thread(() -> System.out.println(Env.EVN_STR)).start();
        System.out.println(Constants.CONSTANTS_STR);
        System.out.println("Main end");
    }

    public static final class Env {
        static final StringBuilder EVN_STR = new StringBuilder("env str");

        static {
            System.out.println("Env static block begin, " + Thread.currentThread());
            // deadlock by Constants
            System.out.println("Env refer -> " + Constants.CONSTANTS_STR);
            System.out.println("Env static block end");
        }
    }

    public static class Constants {
        static final StringBuilder CONSTANTS_STR = new StringBuilder("constants str");

        static {
            System.out.println("Constants static block begin, " + Thread.currentThread());
            // deadlock by Env
            System.out.println("Constants refer -> " + Env.EVN_STR);
            System.out.println("Constants static block end");
        }
    }
}
```

### 原理

 见 [https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html#jls-12.4.2      ](https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html#jls-12.4.2)

**For each class or interface C, there is a unique initialization lock LC** : 

1. Synchronize on the initialization lock, LC, for C. This involves waiting until the current thread can acquire LC.
2. If the Class object for C indicates that initialization is in progress for C by some other thread, then release LC and block the current thread until informed that the in-progress initialization has completed, at which time repeat this step.
3. If the Class object for C indicates that initialization is in progress for C by the current thread, then this must be a recursive request for initialization. Release LC and complete normally.
4. If the Class object for C indicates that C has already been initialized, then no further action is required. Release LC and complete normally.