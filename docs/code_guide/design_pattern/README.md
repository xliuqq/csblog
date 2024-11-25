# 设计模式

## 创建型模式

**抽象实例化的过程**，**类创建型模式**使用继承改变被实例化的类，**对象创建型模式**将实例化委托给另一个对象。

创建型模式在**什么**被创建，**谁**创建它，它是**怎么**被创建的，以及**何时**创建方面有很大的灵活性。

- [工厂模式（Factory）](factory.md)
- [抽象工厂（Abstract Factory）](./abstract_factory.md)
- [构造者模式（Builder）](./builder.md)
- [单例模式（Singleton）](./singleton.md)

通常，设计以使用***Factory Method*开始**，并且当设计者发现需要更大的灵活性时，设计便会**向其它创建型模式演化**。

  

## 结构型模式

- [适配器（Adapter）](adapter.md)
- [桥接（Bridge）](./bridge.md)
- [组合（Composite）]()
- [装饰（Decorator）](./decorator.md)
- [外观（Facade）](./facade.md)
- [享元（Flyweight）](./flyweight.md)
- [代理（Proxy）](./proxy.md)



模式相关性：

- 对象适配器模式跟桥接模式类似，但是**桥接模式目的是将接口部分和实现部分分离**，较为容易并相对独立进行改变；**适配器模式则是改变一个已有对象的接口**。
- **装饰者模式增强其他对象的功能而同时不改变它的接口**，其应用程序的透明性比适配器好。
- 代理模式在**不改变它的接口的条件下，为另一个对象定义了一个代理**。
- **装饰可以改变对象的外表**，而**策略模式可以改变对象的内核**，这是改变对象的两种途径。
- 可以将装饰视为退化、仅一个组件的组合，但装饰的目的不在于对象聚集，而是添加额外的职责。
- Flyweight经常和Composite模式结合起来，用共享叶节点的有向无环图实现一个逻辑上的层次结构；
- 最好**用Flyweight实现State模式和策略模式**；



## 行为型模式

- [职责链（chain）](./chain.md)

- [命令（command）](./command.md)

- [解释器（interpret）](./interpret.md)

- [迭代器（iterator）](./iterator.md)

- [中介者（medium）](./medium.md)

- [备忘录（memento）](./memento.md)

- [观察者（observer）](./observer.md)

- [状态（state）](./state.md)

- [策略（strategy）](./strategy.md)

- [模板方法（template）](./template.md)

- [访问者（visitor）](./visitor.md)

  

## [分布式系统应用设计（容器设计模式）](container_distributed.md)

- 单节点模式：
  - 边车模式
  - 大使模式
  - 适配器模式
- 服务（多节点）模式：
  - 复制服务
  - 分片服务
  - 分散/聚集模式
  - 函数和事件驱动
  - 所有权选举

- 批处理计算模式：
  - 工作队列系统
  - 事件驱动的批处理
  - 协调批处理