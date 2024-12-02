# VarHandle

## 背景

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