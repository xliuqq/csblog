# SPI

SPI 全称为 (Service Provider Interface) ，是**JDK内置的一种服务提供发现机制**，在运行的时候动态寻找实现类。

SPI机制的约定：

- 在**META-INF/services/**目录中创建以**接口全限定名命名**的文件，该文件内容为Api具体**实现类的全限定名**；
- 使用**ServiceLoader类**动态加载META-INF中的实现类；
- 若SPI的实现类为Jar则需要放在主程序**classPath**中；
- Api具体实现类必须有一个**不带参数的构造**方法；

**基于接口的编程＋策略模式＋配置文件**组合实现的动态加载机制。

案例：

- 数据库驱动加载；
- Slf4j具体的日志实现加载；
- Spring，Dubbo等。



虽然ServiceLoader也算是使用的延迟加载，但是基本只能通过遍历全部获取，也就是**接口的实现类全部加载并实例化一遍**。

如果多个实现类，需要自行选择哪种实现（一般是在接口中定义函数，可以接受哪种的schema）。



## 使用

```java
// 默认使用 线程的 classloader
final Iterator<T> iterator uploadCDN = ServiceLoader.load(UploadCDN.class).iterator();
// 可以定义多个实现，如果某个实现的类的依赖未引入（会报错），则继续下一个
while (iterator.hasNext()) {
    try {
        return iterator.next();
    } catch (ServiceConfigurationError ignore) {
        // ignore
    }
}
return null;
```

