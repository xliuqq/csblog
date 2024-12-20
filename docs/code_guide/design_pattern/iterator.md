# 迭代器模式

## 意图

提供一种方法顺序访问一个聚合对象中各个元素, 而又无须暴露该对象的内部表示。。

**解决：**不同的方式来遍历整个整合对象。

## 适用性

Java的 Iterator 遍历一个聚合对象。

1、访问一个聚合对象的内容而无须暴露它的内部表示。

 2、需要为聚合对象提供多种遍历方式。

 3、为遍历不同的聚合结构提供一个统一的接口。



## 类图

![迭代器模式的结构图](.pics/iterator/iterator.png)

1. **抽象聚合**（Aggregate）角色：定义存储、添加、删除聚合对象以及创建迭代器对象的接口。
2. **具体聚合**（ConcreteAggregate）角色：实现抽象聚合类，返回一个具体迭代器的实例。
3. **抽象迭代器**（Iterator）角色：定义访问和遍历聚合元素的接口，通常包含 hasNext()、first()、next() 等方法。
4. **具体迭代器**（Concretelterator）角色：实现抽象迭代器接口中所定义的方法，完成对聚合对象的遍历，记录遍历的当前位置。



## 优缺点

**优点：** 1、它支持以不同的方式遍历一个聚合对象。 2、迭代器简化了聚合类。 3、在同一个聚合上可以有多个遍历。 4、在迭代器模式中，增加新的聚合类和迭代器类都很方便，无须修改原有代码。

**缺点：**由于迭代器模式将存储数据和遍历数据的职责分离，增加新的聚合类需要对应增加新的迭代器类，类的个数成对增加，这在一定程度上增加了系统的复杂性。