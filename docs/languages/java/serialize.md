# 序列化

> [示例项目](https://gitee.com/oscsc/jvm/tree/master/serialize)

序列化 (Serialization)是将对象的**状态信息**转换为可以**存储或传输**的形式的过程。

- 特定语言：如 JDK 序列化，Kryo 序列化；
- 跨语言：如 Hessian, Protobuf, Thrift, Avro, Json 等；

## JDK

序列版本**UID（serial version UID）**，流的唯一标识符：

- **默认根据类名称、实现接口、成员等信息自动生成**，因此很容易出现兼容性问题；
- `transient`修饰的变量，默认序列化不会进行序列化；



### 自定义序列化

#### readObject & writeObject

Java实现序列化，实现`Serializable`接口，该接口中没有任何的方法，但是要实现自定义的序列化方法时，需要重载`readObject`, `writeObject`且都是private

- 当`ObjectOutputStream`调用`writeObject`时，会根据`instanceof Serializable`，以及类的`ObjectStreamClass`（硬编码读取该三个方法），判断是否调用用户实现的方法还是默认的序列化方法；


```java
private void writeObject(java.io.ObjectOutputStream out) throws IOException
private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException;
```

除了基本的`writeObject`和`readObject`之外，以下三个函数也可以控制序列化和反序列化：

```java
// readObjectNoData is used in an unusual case where the serializer (writer) is working with a version of a class with no base class, whereas the deserializer (reader) of the class has a version of the class that IS based on a subclass. The subclass can say "it's ok if my base class isn't in the serialized data - just make an empty one" by implementing readObjectNoData.
// 序列化时用旧类，反序列化用新版本类（且变成子类），可以在父类上定义该方法，生成默认的父对象的成员值
private void readObjectNoData() throws ObjectStreamException;

// 实际序列化的对象将是作为writeReplace方法返回值的对象，而且序列化过程的依据是实际被序列化对象的序列化实现
private Object writeReplace()

// 再readObject()调用之后自动调用， 将反序列化之后的对象替换掉（可用于单例/枚举等场景）
Object readResolve() throws ObjectStreamException;

```



#### writeReplace & readResolve

Serializable还有两个标记接口方法可以实现序列化对象的替换，即`writeReplace`和`readResolve`:

- `Object writeReplace() throws ObjectStreamException`
  - 序列化时会**先调用`writeReplace`方法将当前对象替换成另一个对象**（该方法会返回替换后的对象，替换后的对象必须可序列化）并将其写入流中，调用过程为 `writeObject(writeReplace())`
- `Object readResolve() throws ObjectStreamException`
  - **`readResolve`会在`readObject`调用之后自动调用**，`readResolve`里再对该对象进行一定的修改，而最终修改后的结果将作为`readObject`的返回结果；
- 应用就是**保护性恢复单例**的对象
  - 对单例对象，重写`readResolve`方法，保证返回的是系统中唯一的单例。

### 枚举类型的序列化

在枚举类型的序列化和反序列化上，Java做了特殊的规定，即**在序列化的时候Java仅仅是将枚举对象的name属性输出到结果中，反序列化的时候则是通过java.lang.Enum的valueOf方法来根据名字查找枚举对象**。

同时，**编译器是不允许任何对这种序列化机制的定制**，因此禁用了writeObject、readObject、readObjectNoData、writeReplace和readResolve等方法。

### Lambda 表达式的序列化（TODO）

```scala
val writeReplace = closure.getClass.getDeclaredMethod("writeReplace")
writeReplace.setAccessible(true)
writeReplace.invoke(closure).asInstanceOf[java.lang.invoke.SerializedLambda]
```

lambda 会生成如下字节码

```java
// lambda生成的字节码
@FunctionalInterface
interface Print<T> {
    public void print(T x);
}
public class Lambda {  
    public static void PrintString(String s, Print<String> print) {
        print.print(s);
    }
    private static void lambda$0(String x) {
        System.out.println(x);
    }
    final class $Lambda$1 implements Print{
        @Override
        public void print(Object x) {
            lambda$0((String)x);
        }
    }
    public static void main(String[] args) {
        PrintString("test", new Lambda().new $Lambda$1());
    }
}

```



## [Kryo](](https://github.com/EsotericSoftware/kryo))

> Java 主流的序列化框架。

###  线程

> [非线程安全]([GitHub - EsotericSoftware/kryo: Java binary serialization and cloning: fast, efficient, automatic](https://github.com/EsotericSoftware/kryo#thread-safety))，`Each thread should have its own Kryo, Input, and Output instances.`

使用  ThreadLocal 或者 Pool 。



## [Hessian](http://hessian.caucho.com/doc/hessian-serialization.html#toc)

> 跨语言，一种动态类型、二进制序列化和 Web 服务协议，专为面向对象的传输而设计

**注意子类和父类不能有同名字段，子类的属性总是会被父类的覆盖**：

- 当序列化对象是一个对象继承另外一个对象的时候，当一个属性在子类和有一个相同属性的时候，反序列化后子类属性总是为null。