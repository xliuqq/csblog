# 装饰器模式

## 意图

**动态地给一个对象添加一些额外的职责**。就增加功能来说，Decorator模式相比生成子类更灵活。



## 适用性

- 在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责；
- 处理那些可以撤销的职责；



## 类

Decator继承Component，使用Component对象，并添加额外的方法。（实例：Java种的流）

<img src=".pics/decorator/o_decorator-class-diagram.jpg" alt="装饰者模式（Decorator）结构图" style="zoom:80%;" />

1. 抽象构件(Component)角色：给出一个抽象接口，以规范准备接收附加责任的对象。
2. 具体构件(ConcreteComponent)角色：定义一个将要接收附加责任的类。
3. 装饰(Decorator)角色：持有一个构件(Component)对象的实例，并定义一个与抽象构件接口一致的接口。
4. 具体装饰(ConcreteDecorator)角色：负责给构件对象“贴上”附加的责任。



## 优缺点

优点：

- 装饰模式可以提供比继承**更多的灵活性**。装饰模式允许系统动态决定“贴上”一个需要的“装饰”，或者除掉一个不需要的“装饰”。继承关系则不同，继承关系是静态的，它在系统运行前就决定；
- 避免在层次结构高层的类有太多的特征；
- 比使用继承关系需要**较少数目的类**。

**缺点**：

- 使用装饰模式会产生比使用继承关系**更多的对象**，多的对象会使得查错变得困难，特别是这些对象看上去都很相像。



## 示例

Java源码中文件流的相关设计（多种读取、性能方式的组合）：

- java.io.BufferedReade（Decorator Implementation）;  
- java.io.FileReader（ConcrentComponent）;   
- java.io.Reader（Component）;

AOP，切面的实现，也称为装饰器。