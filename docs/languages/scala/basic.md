# 基础

## ==, eq, equals

根据[官方API](http://www.scala-lang.org/api/current/#scala.AnyRef)的定义：

```scala
// The expression x == that is equivalent to if (x eq null) that eq null else x.equals(that).
final def ==(arg0: Any): Boolean
            
// Tests whether the argument (that) is a reference to the receiver object (this).
final def eq(arg0: AnyRef): Boolean
            
// The equality method for reference types.
def equals(arg0: Any): Boolean
```

简言之，**equals方法是检查值是否相等**，而**eq方法检查的是引用是否相等**，**==判断的是值相等或者引用相等**

 

Array() 新建出来的对象的equals方法是Object的方法，要通过sameElements方法， arr1 sameElements arr2



## sealed

修饰class或trait：**不能在类定义的文件之外定义新的子类**



## Option-Either-Try

Option：Option是抽象类，**Some**和**None**是其子类实现，用来**防止空指针调用**的问题，且**无需多余的 if 判断**。

Try：对于**可能抛出异常的操作**，使用Try进行封装，得到Try的**子类 Success 或 Failure** 。

Either：表示不同输入会得到两个不同的类型 Left， Right。



## case-class

样例类（case class）适合用于**不可变**的数据。它是一种特殊的类，能够被优化以用于模式匹配。

- 创建 case class 和它的伴生 object
- 实现了 apply 方法让你不需要通过 new 来创建类实例
- 默认为主构造函数参数列表的所有参数前加 val
- 添加天然的 hashCode、equals 和 toString 方法。由于 == 在 Scala 中总是代表 equals，所以 case class 实例总是可比较的
- 生成一个 `copy` 方法以支持从实例 a 生成另一个实例 b，实例 b 可以指定构造函数参数与 a 一致或不一致
- 由于编译器实现了 unapply 方法，一个 case class 支持模式匹配



## case-object

类中有参和无参，当类有参数的时候，用case class ，当类没有参数的时候那么用case object。

