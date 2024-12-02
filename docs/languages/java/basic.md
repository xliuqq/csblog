# 基础知识

- **对象的默认初始值都是null**，包括String，new String[]；

- **Iterator的删除元素方法是remove**，不会产生concurrentModify异常；

- **HashMap的key，value可以为null**，但是HashTable和ConcurrentHashMap都不可以；

- instanceof，多维数组是Object[]的类型，但不是int[]的类型，int[]是Object类型，但不是Object[]类型，基本类型也不是Object类型；

- System.**getProperty**("java.library.path") 参数可以通过**-D配置**，System.getEnv中获取当前环境的环境变量；

  - user.dir：**当前用户工作目录**，即当前执行java命令的路径

- **hashcode 和 equals方法**：

  1）如果o1.equals(o2)，那么o1.hashCode() == o2.hashCode()总是为true的。

  2）如果o1.hashCode() == o2.hashCode()，并不意味着o1.equals(o2)会为true。

- Java 对 **Interger的-128~127的对象进行缓存**，返回new Integer(321)是两个不同的对象；

- Thread.currentThread.getStackTrace()可以获得**当前调用栈**的数组StackTraceElement[]，可以获取方法名，文件名，行数等信息；

- StringBuffer被隐式使用，当String使用+操作时，显示使用StringBuilder或StringBuffer可以提升性能；StringBuffer线程安全；

- Java的**final字段不会自动类型提升**；private和static方法无法被重载；

- Thread状态getState, isAlive，**标记为DEAD状态的线程可能复活，不会被垃圾回收**；

- **双大括号可以用来实现实例的创建**（生成匿名类，然后将第二个括号里内容附在匿名类里），**单个大括号实现匿名类**；

  ```java
  // 慎用，匿名类会包含一个外围实例，造成内存泄漏的问题
  Set<Integer> set = new HashSet<Integer>() {{
      add(1);
      add(2);
  }};
  ```

- final修饰符的作用

  - 类：无法被继承
  - 方法：无法被子类覆盖
  - 变量：无法被重新赋值

- 获取到**类路径（classpath）下的资源路径**信息

  - ClassLoader.getResource(String classpathFilePath)，类似“双亲委派机制”，以 / 开头的资源是在ClassLoader目录下的资源，并非指的是当前应用的类加载路径下的资源。<font color="red">如果资源是位于classpath下的不要以 / 开头</font>；
  - Class<?>.getResource(String  classpathFilePath)，当路径信息不以 / 开头时，获取的是相对于当前类所在路径的相对资源；<font color="red">当路径信息以 / 开头时，则获取的是当前应用类加载路径下的资源</font>

- jar中的文件的获取方式：

  - 通过类装载器来来获定位资源

    `Icon myIcon = new ImageIcon(getClass().getResource("/com/directory.gif"))`

- Path用来获取路径：

  - Path.toAbsolutePath()获取**绝对路径**；
  - File.getCanonicalPath()获取**规范绝对路径**，将绝对路径中的".."和"."转换消除后的得到的路径

  - Path.toRealPath()获取**实际路径**，相对地址转绝对地址，.. 和 . 转为规范路径，符号链接转为实际路径；

- Runtime Exception 无需被 try-catch 或方法加声明；



## 函数

- Java重载的方法（同名但签名不一样的函数）的匹配是**按照声明类型匹配形参类型**，而不是靠实际类型；


**lombok.NonNull、javax.annotation.Nonnull** 和 **javax.validation.constraints.NotNull**的区别和使用场景：

- - **@NonNull**用在强制入参非空、属性非空的场景下，编译时会生成空指针检查的代码，如果不满足，则会抛出NullPointerException。
  - **@Nonnull**用在返回结果不为空的场景，主要由IDE识别并给出告警提示。
  - **@NotNull**主要用在Validator校验场景下，限制参数非空，如果不满足，则校验失败。



## 异常与日志

> Error 是不可控的错误，程序无法处理，必须人工介入；

Exception 分为：

- 受检异常：需要显式处理（try catch 或者 throws）；
- 非受检异常（`RuntimeException`子类）

finally 只用来清理资源、释放连接、关闭流等操作：

- 不要在 finally 代码块中赋值，catch 中的 return 会先暂存，再执行 finally 中的代码；
- 不要在 finally 中使用 return 语句，会使返回值的判断变得复杂；

### try 跟 lock 

- **try 代码块前调用 lock() 方法**，避免由于加锁失败导致 finally 调用 unlock 抛出异常；
  - 对未加锁的对象解锁会抛出 unchecked 异常，导致**加锁失败的原因被覆盖**；

```java
Lock lock = new XXXLock();
lock.lock();
try {
    // do something
} finally {
    lock.unlock();
}
```



### 异常和错误码

推荐规范

- **对外提供的开放接口使用错误码**；
- **内部开放的接口考虑使用 Result 对象**封装错误码和错误描述信息；
  - 若调用方未捕获，则会产生运行时错误，导致程序中断；
- **应用内部直接抛出异常对象**；



### 返回 NULL

> 契约式变成理念处于**防御式编程**理念的下风。

防止 NPE 一定是调用方的责任（接收方服务可能会异常）。



## 类和接口

### 基本知识

- 子类默认调用的是父类的无参构造函数，只有类没有构造函数时才会默认构造；
- 覆盖父类的方法时，访问权限不能变小（public > package > protected > private)；
- 接口默认就是abstract public，其成员函数默认是abstract public，其成员变量默认是static final；
- Java中**静态绑定**的情况：**调用类方法；初始化方法；私有方法**；其他都是动态绑定；
- **this()和super()都必须在构造函数的第一行**，不会同时出现（先初始化再使用成员）；
- 内部类的实例创建：`Outer.Inner in = new Outer().new Inner()`；静态内部类的创建：`Outer.Inner in = new Outer.Inner()`；
- 内部类持有外部类，但外部类关闭时，**内部类依然被持有会造成泄漏**；



### 函数式接口（JDK8）

- 一个函数式接口只有一个抽象方法(SAM，single abstract method)；
- Object 类中的 public abstract method 不会被视为单一的抽象方法；
- 函数式接口可以有默认方法和静态方法；
- 函数式接口可以用@FunctionalInterface 注解进行修饰。

### String

- Java中字符串采用**Unicode编码**，一个char占两个字节；

  - JDK8后，String 内部采用 `byte[]` 而不是 `char[]`

- `\u`是unicode编码；

  - Java String 采用 [UTF-16编码](../unicode.md)；
  - String的 length 方法返回的是Unicode的 code units 的数量；

  ```java
  // name.getBytes().length == 15 != (5 == name.length())
  String name = "我是中国人";
  System.out.println(name.getBytes().length + ":" + name.length());
  ```

  - String中的**charAt**和**codePointAt**方法；
  - 非基本平面的String遍历方式

  ```java
  // JDK 8, CharSequence#codePoints了返回IntStream
  "str".codePoints().forEach(c -> ...);
  ```

  String 的 **getBytes**方法，是将字符串转换为特定编码规则后的字节数组

  - 默认的是Charset.defaultCharset()；

- String的带Charset的构造方法，Charset指的是byte[]的编码方式；

- **InputStreamReader读取二进制文本数据流时，可以指定Charset；**

- String ”main"的内存占用图示：

<img src=".pics/basic/java_string_memory.png" alt="String memory" style="zoom:50%;" />

### Files

> 更多场景，推荐使用 apache common io.

`java.nio.Files` 类，提供的文件的常见操作，如 创建、遍历、存在、删除、拷贝等静态方法；

- Files.list 方法只遍历当前目录下的文件和目录，`Files.walk` 方法遍历指定目录下的所有文件和目录，除非指定了最大深度



## 泛型

`<?>`称为通配符集合，**可以接受任何类型的集合引用赋值**，**不能调用参数是泛型的相关函数**；

```java
List<?> stus = new ArrayList<>();
// 编译错，因为 add 方法的参数是泛型，但是 ? 不代表任何的数据类型
stus.add(1);
```

`<? extend T>`：赋值给**任何 T 及 T子类**的集合，***Get First***，用于消费集合元素，

- 不能调用参数是泛型的相关函数，除 null 之外；
- **get 返回的类型是 T 及 T的父类**；

`<? super T>`：赋值给**任何 T 及 T的父类**的集合，***Put First***，用于生产集合元素；

- 所有的元素类型都是 T 的父类，**put 元素类型为 T及其子类**
- 调用返回类型是泛型的函数，其返回结果是 Object；

```java
List<? extends Father > extendFather = new ArrayList<>();
List<? super Father > superFather = new ArrayList<>();

Father cat = extendFather.get(1);
Object object = superFather.get(0);
superFather.add(new Son());
```



### Iterator

- Java中非线程安全的集合如ArrayList等，其Iterator是**fail-fast机制**，iterator遍历的时候，不可以进行添加或删除操作；

  - 遍历时删除，使用 iterator 的 remove方法

  ```java
  // 单线程遍历时删除，需要通过 Iterator的remove方法操作
  Iterator<String> iter = list.iterator();
  while (iter.hasNext()) {
      String item = iter.next();
      if (shouldRemove(item)) {
          iter.remove();
      }
  }
  ```

  

- 对于线程安全的如ConcurrentHashMap而言，Iterator在遍历底层数组。在遍历过程中，如果已经遍历的数组上的内容变化了，迭代器不会抛出ConcurrentModificationException异常。**如果未遍历的数组上的内容发生了变化，则有可能反映到迭代过程中**。这就是ConcurrentHashMap迭代器**弱一致**的表现。



### Java Time

```java
long t1 = System.currentTimeMillis();
long t2 = new Date().getTime();
long t3 = Calendar.getInstance().getTimeInMillis();
```

将提供自unix纪元以来的毫秒数，这是一个固定的时间点，不受本地时区影响；

**GMT**（Greenwich Mean Time，格林威治时间)是时区时间。

**UTC**（Coordinated Universal Time协调世界时）是个标准时间；UTC作为一个标准时间和各个时区时间换算

LocalDateTime 是没有时区概念的时间，ZonedDateTime是有时区概念的时间。

LocalDateTime.now(ZoneId.of("UTC")) 是根据UTC构建时间，默认是根据系统的时区创建时间；



## Stream

JDK 8 引入，对于集合数据可以流式处理。

### 创建

Stream.of 可以创建Stream，集合类通过通过stream方法或者parallelStream创建流。

### 中间操作

peek, skip, sorted, map, flatMap, filter, distinct

### 末端操作

collect, count, reduce

max, min

forEach, forEachOrdered

findFirst, findAny

anyMatch, allMatch, noneMatch

toArray

### Collectors

groupingBy：按Key聚合

groupingByConcurrent

joining

partitioningBy

maxBy, minBy

toList, toMap, toSet, toConcurrentMap, toCollection



## Flow

JDK 9 中的反应流。

Flow.Publisher 接口：实现信息的发布者；

Flow.Subscriber接口：实现信息的订阅者；

Flow.Subsciption接口：发布和订阅的订阅关系；



## 模块化

Java 9 之后引入**模块化的概念**。

- 使用 public 修饰的类只能在当前模块使用；



## 加密解密

Java API

- `Cipher.getInstance(transformation)`，“算法/模式/填充”或“算法”，如 AES/CBC/PKCS5Padding；
  - PKCS7Padding：PKCS #7 填充字符串由一个字节序列组成，每个字节填充该填充字节序列的长度，0x01-0x08，根据最后一位知道填充字节数；
  - PKCS5Padding的blocksize为8字节，PKCS7Padding的blocksize可以为1到255字节；





## 容器

### CPU/Memory

使用 java 8u191+ 或者 JDK 10之后的版本，默认开启`+XX:-UseContainerSupport`；

- JVM 知道在Docker容器中运行，从主机读取cgroup限制（已分配给容器的CPU数量和总内存），而不是从宿主机提取；
- `-XX:InitialRAMPercentage`：设置JVM使用容器内存的初始百分比。建议与 -XX:MaxRAMPercentage 保持一致，推荐设置为70.0。
- `-XX:MaxRAMPercentage`：其值介于0.0到100.0之间，默认值为25.0。设置JVM使用容器内存的最大百分比。由于存在系统组件开销，建议最大不超过75.0，推荐设置为70.0。