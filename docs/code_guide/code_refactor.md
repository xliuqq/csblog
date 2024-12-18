# 重构

> 以下内容部分来源于书籍 《重构：改善既有代码的设计》



## 基本要求

| 类型   | 要求                     |
| ------ | ------------------------ |
| 变量名 | **说明变量代表的含义**   |
| 函数名 | **说明函数内部做的事情** |
| 类名   | **说明类的功能**         |



## 代码坏味道

### 基本

#### 重复代码

**检查：**IDEA有提示

**重构：**抽取出同样代码为函数（放到新类或者提取到父类中），进一步可以使用***模板方法***的设计模式。



### 函数

#### 过长的函数(Long Method)

**检查：**配置CheckStyle等规则，一般在80行以内

**重构：**代码切分，抽象出逻辑层次一致的子函数（如`if/else`代码块可以提炼成函数）

#### 过长的参数(Long Parameter List)

**检查：**配置CheckStyle等规则，一般不超过5个

**重构：**多个参数用参数对象/数据类（*POJO*）进行替代



### 类

#### 过大的类(Large Class)

**检查：**文件长度过大？（不是很精确）

**重构：**单一职责对类的功能进行划分，拆分成多个类

#### 发散式变化(Divergent Change)

**检查：**类因为不同方向的原因变化而修改

**重构：**单一职责对类的功能进行划分，拆分成多个类

#### 霰弹式修改(Shotgun Surgery)

**检查：**修改某个功能，要在不同的类中做出愈多小修改

**重构：**把所有需要修改的代码放进同一个类中（需要与***发散式变化***进行权衡）

**目标：**使**外界变化** 与 **需要修改的类** 趋于**一一对应**

#### 依恋情结(Feature Envy)

> 将数据和加诸其上的操作行为包装在一起

**检查：**某个函数为了计算某值，从另一个对象那儿调用几乎半打的取值函数（getting method）

**重构：**判断哪个class拥有最多「被此函数使用」的数据，然后就把这个函数和那些数据摆在一起

**说明：**策略模式、访问者模式、代理模式，虽然违法该规则，但是是为了解决*发散式变化*

#### 数据泥团(Data Clumps)

**检查：**两个类中有很多相同的字段，函数签名中有相同的参数

**重构：**「总是绑在一起出现的数据」真应该放进属于它们自己的对象中

#### 基本类型偏执(Primitive Obsession)

**检查：**对于一些有物理意义的变量，使用基本类型，如起始值、地址（String）

**重构：**通过小对象重构，类名表示具体的物理含义，比如Range，Address（里面可能就一个String）

#### Switch语句

**检查：**多处switch语句，同样的枚举字段（或`type code`）判断

**重构：**最多一处switch语句（或者将switch换成反射），然后用多态定义不同类型的不同实现

#### 平行继承体系(Parallel Inheritance Hierarchies)

**检查：**为某个class增加一个subclass，必须也为另一个class相应增加一个subclass

**重构：**让一个继承体系的实体（instance）引用另一个继承体系的实体

#### 冗赘类(Lazy Class)

**检查：**一个类用处少或者已经无用

**重构：**删除或者将其加入调用类中(Inline Class)，如果是某些子类没有做足够的工作，使用Collapse Hierarchy (折叠继承体系)

#### 夸夸其谈未来性(Speculative Generality)

**检查：**当前用不到，以未来会使用为借口

**重构：**删除，如果某个abstract class没有太大作用，请运用Collapse Hierarchy(折叠继承体系)

#### 临时字段(Temporary Field)

**检查：**类中变量仅为某种特定情势而设（从类的生命周期），比如只参与到某个算法函数中

**重构：**变量和其相关函数提炼到一个独立class中，提炼后的新对象将是一个method object

#### 过度耦合的消息链(Message Chains)

**检查：**`a.getB().getC().getD()`

**重构：**委托函数，与`中间人问题`是个权衡

**说明：**[迪米特法则]()

#### 中间人(Middle Man)

**检查：**某个class接口有一半的函数都委托给其他class

**重构：**Remove Middle Man，直接和实责对象打交道

#### 狎昵关系(Inappropriate Intimacy)

**检查：**两个classes过于亲密，花费太多时间去探究彼此的private成分

**重构：**

- 将双向关联改为单向关联（Change Bidirectional Association to Unidirectional）。

- 提炼类，将两个类的共同点提炼到新类中，让它们共同使用新类。

- 继承往往造成过度亲密，运用以委托取代继承（Replace Inheritance with Delegate）

#### 异曲同工的类(Alternative Classes with Different Interfaces)

**检查：**两个函数做同一件事，却有着不同的签名式（signatures）

**重构：**抽象并统一

#### 不完美的程序库类(Incomplete Library Class)

**检查：**外部的类库无法满足实际的需求，需要额外的方法

**重构：**

- 修改类库的一两个函数 - 引入外部函数（Introduce Foreign Method）
- 添加一大堆额外行为 - 添加本地扩展（Introduce Local Extension）

#### Data Class(纯稚的数据类)

**检查：**数据类把全部字段单纯的通过getter/setter暴露出来

**重构：**

- 不该被其他classes修改的值域，请运用 Remove Setting Method。
- 暴露抽象接口（将调用行为代码移到Data Class中），封装内部结构。

#### 被拒绝的遗赠(Refused Bequest)

**检查：**1）子类继承父类的所有函数和数据，子类只挑选几样来使用；2）子类只复用了父类的行为，却不想支持父类的接口

**重构：**1）为子类新建一个兄弟类，再运用下移方法（Push Down Method）和下移字段（Push Down Field）把用不到的函数下推个兄弟类；

2）运用委托替代继承（Replace Inheritance with Delegation）来达到目的。

#### 过多的注释(Comments)

**检查：**需要注释来解释一块代码做了什么

**重构：**抽象代码实现为函数，通过好的命名方式阐述代码的功能



## 重新组织函数

### 提炼函数（Extract Method）

> 一段代码可以被组织在一起并独立出来。

函数深度（层次）应该保持统一级别。

```java
void printOwing(double amount) {
    printBanner(); 			// 同一调用层次级别
   	System.out.println("amount" + amount); // 不是同一调用层次
    printDetailer();		// 同一调用层次级别
}
```

